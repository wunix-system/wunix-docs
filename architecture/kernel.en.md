[ 🇷🇺 Русский ](kernel.ru.md) | [ 🇬🇧 English ](kernel.en.md)

# Wunix Core Kernel Architecture

**Wunix Core** is a hybrid modular kernel targeting the `x86_64` architecture, operating in 64-bit Long Mode under the modern **Limine** boot protocol.

---

## 1. Address Space Layout (Higher-Half Model)

The Wunix kernel loads into the upper half of the 64-bit virtual address space (Higher-Half). This reserves the entire lower half of memory (`0x0000_0000_0000_0000` through `0x0000_7FFF_FFFF_FFFF`) for future Ring 3 userland processes.

```
0xFFFF_FFFF_FFFF_FFFF ┌────────────────────────────────────────┐
                      │ Base Kernel Code (Limine Mapping)      │
0xFFFF_FFFF_8000_0000 ├────────────────────────────────────────┤
                      │ Kernel Dynamic Heap                    │ (1+ MiB)
0xFFFF_9000_0000_0000 ├────────────────────────────────────────┤
                      │ Higher-Half Direct Map (HHDM)          │
                      │ (Direct physical memory window)        │
0xFFFF_8000_0000_0000 ├────────────────────────────────────────┤
                      │ Non-canonical Hole                     │
0x0000_7FFF_FFFF_FFFF ├────────────────────────────────────────┤
                      │ User Space (Future Ring 3 processes)   │
0x0000_0000_0000_0000 └────────────────────────────────────────┘
```

---

## 2. Modular Subsystem Layout

In accordance with kernel engineering standards ([`AGENTS.md`](file:///home/valdis/my-os/AGENTS.md)), kernel source code is partitioned into isolated subsystems:

```
wunix/kernel/src/
├── arch/x86_64/              # Low-level CPU hardware architecture code
│   ├── gdt.rs                # GDT, TSS, and emergency IST0 stack (20 KB) for Double Fault
│   ├── idt.rs                # Interrupt Descriptor Table and CPU exception handlers
│   ├── interrupts.rs         # Hardware interrupt dispatching (IRQ)
│   ├── pic.rs                # 8259 PIC controller remapping (vectors 32..47)
│   ├── timer.rs              # PIT 8254 system timer (100 Hz, tick counter, uptime)
│   └── port.rs               # Safe hardware I/O port abstractions (inb, outb)
├── drivers/                  # Hardware device drivers
│   ├── display.rs            # Limine framebuffer graphical console
│   ├── font.rs               # 8×16 bitmap font rasterizer (ASCII + Cyrillic)
│   ├── serial.rs             # COM1 UART 16550 serial driver for diagnostics
│   └── keyboard.rs           # PS/2 controller, ring buffer, layout switcher
├── memory/                   # Memory management subsystem
│   ├── frame.rs              # 4 KiB physical frame allocator utilizing Limine memmap
│   ├── paging.rs             # 4-level page table management and virtual translation
│   └── heap.rs               # Dynamic kernel heap (`alloc`, `linked_list_allocator`)
├── fs/                       # Virtual File System (VFS) and storage subsystem
│   ├── path.rs               # Path engine (C:\ -> /win/drives/c/), slash normalization, case-insensitivity
│   ├── ramfs.rs              # In-memory FS powered by spin::RwLock, files, directories, and symlinks
│   ├── tar.rs                # POSIX ustar initramfs unpacker
│   ├── vfs.rs                # Inode, FileSystem traits, Metadata, DirEntry structures, VfsError
│   └── mod.rs                # Root mounting ('/'), path resolution, Unix/NT skeleton, unified profile symlink
├── task/                     # Multitasking subsystem (in development)
│   ├── context.rs            # CPU register context state
│   └── scheduler.rs          # Kernel thread scheduler
├── i18n/                     # Localization subsystem (kernel/src/i18n)
│   └── mod.rs                # Language state, MsgKey catalog, tr(), t(), cmdline parser
├── lib.rs                    # Kernel subsystem initialization entry (kernel_init)
└── main.rs                   # Kernel entry point (_start) and panic handling
```

---

## 3. Memory Management Subsystem

Memory initialization proceeds across three distinct phases:

1. **HHDM and Page Translation (`memory/paging.rs`)**:
   Limine provides the Higher-Half Direct Map offset via `hhdm_request`, giving the kernel immediate virtual access to any physical address by adding the HHDM base.
2. **Physical Frame Allocator (`memory/frame.rs`)**:
   Parses `memmap_request` from Limine, identifies accessible memory regions (`LIMINE_MEMMAP_USABLE`), and dispenses free physical 4096-byte frames.
3. **Kernel Heap (`memory/heap.rs`)**:
   Positioned at virtual address `0xFFFF_9000_0000_0000`. Initial capacity is 1 MiB, enabling global Rust allocator integration and dynamic collection types (`Box`, `Vec`, `String`, and `format!`).

---

## 4. Interrupts and Hardware Bring-Up

1. **GDT and TSS (`arch/x86_64/gdt.rs`)**:
   Configures 64-bit kernel code and data segment descriptors. Allocates a dedicated 20 KB emergency stack (IST0). If a Double Fault occurs (e.g., due to stack overflow), the CPU hardware automatically switches to IST0, preventing an immediate Triple Fault and CPU reboot.
2. **PIC 8259 Remapping (`arch/x86_64/pic.rs`)**:
   Legacy BIOS interrupt lines are remapped to vector range `32..47`, avoiding conflicts with standard Intel processor exceptions (`0..31`).
3. **IDT (`arch/x86_64/idt.rs`)**:
   Registers safe Rust handlers for processor faults (Breakpoint, Page Fault, General Protection Fault) with register dumps.

---

## 5. Graphical Console and Text Output

- The kernel queries Limine for a high-resolution linear framebuffer (`FramebufferRequest`).
- In [`drivers/display.rs`](file:///home/valdis/my-os/wunix/kernel/src/drivers/display.rs), a console buffer is implemented featuring:
  - Automatic line scrolling when the screen buffer fills;
  - Escape character handling: `\n`, `\r`, `\t`, and `\x08` (Backspace);
  - Hardware cursor and line-clear primitives;
  - High-contrast visual palette (white text on dark background).
- Characters are rendered via the 8×16 monospaced bitmap font engine in [`drivers/font.rs`](file:///home/valdis/my-os/wunix/kernel/src/drivers/font.rs).

---

## 6. VFS Subsystem and Storage Engine

The Virtual File System subsystem ([`fs/mod.rs`](file:///home/valdis/my-os/wunix/kernel/src/fs/mod.rs)) manages a unified kernel namespace tree:

1. **VFS Abstractions (`fs/vfs.rs`)**: `Inode: Send + Sync` and `FileSystem: Send + Sync` traits decouple core kernel logic from specific filesystem on-disk layouts.
2. **In-Memory Storage (RamFS, `fs/ramfs.rs`)**: An in-memory filesystem `RamFs` is mounted as root (`/`), utilizing granular `spin::RwLock` instances embedded directly within `RamInode` objects.
3. **Hybrid Tree and Path Translation (`fs/path.rs`)**: A combined directory skeleton hosts standard Unix directories (`/bin`, `/etc`, `/home`, `/dev`, `/proc`, `/sys`) alongside Windows NT structures (`/win/drives/c/Windows/System32`, `/win/registry`). Paths matching `C:\...` are automatically mapped to `/win/drives/c/...`, and queries within `/win` run case-insensitively.
4. **Unified User Profile**: A symbolic link at `/win/drives/c/Users/user -> /home/user` provides seamless access to user documents and settings across POSIX tools and ported Win32 binaries.
5. **Initramfs Unpacking (`fs/tar.rs`)**: An in-memory TAR archive delivered via the Limine boot protocol (`ModuleRequest`) is parsed as POSIX ustar (512-byte blocks) and populated over the root `RamFs`.

For complete architectural details, see [Virtual File System (VFS) and Hybrid Path Architecture](vfs.en.md).
