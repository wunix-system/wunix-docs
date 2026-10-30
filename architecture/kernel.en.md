[ 🇷🇺 Русский ](kernel.ru.md) | [ 🇬🇧 English ](kernel.en.md)

# Wunix Core Kernel Architecture

**Wunix Core** is a hybrid modular 64-bit kernel designed for the `x86_64` processor architecture, built from the ground up in Rust. The kernel operates in Long Mode under the modern **Limine** boot protocol, isolates userland workloads in Ring 3, and features a dual system call ABI (Linux/POSIX and Windows NT subsystem).

---

## 1. Virtual Address Space Layout (Higher-Half Model)

Wunix loads exclusively into the upper half of the 64-bit canonical virtual address space (Higher-Half). The lower half is completely isolated and reserved for userland processes (Ring 3):

```
0xFFFF_FFFF_FFFF_FFFF ┌──────────────────────────────────────────┐
                      │ Base Kernel Code (Limine Mapping)        │
0xFFFF_FFFF_8000_0000 ├──────────────────────────────────────────┤
                      │ Kernel Dynamic Heap                      │ (1+ MiB, linked_list_allocator)
0xFFFF_9000_0000_0000 ├──────────────────────────────────────────┤
                      │ Higher-Half Direct Map (HHDM)            │
                      │ (Physical memory access window)          │
0xFFFF_8000_0000_0000 ├──────────────────────────────────────────┤
                      │ Non-canonical Hole                       │
0x0000_7FFF_FFFF_FFFF ├──────────────────────────────────────────┤
                      │ Virtual Kernel Library (vDSO / Syslib)   │ (0x0000_7FFF_FFFF_0000)
                      ├──────────────────────────────────────────┤
                      │ Process User Stack                       │ (Grows down from 0x7FFF_... )
                      ├──────────────────────────────────────────┤
                      │ Dynamic Memory & mmap Mappings           │
                      ├──────────────────────────────────────────┤
                      │ ELF / PE User Code & Data (Ring 3)       │
0x0000_0000_0000_0000 └──────────────────────────────────────────┘
```

---

## 2. Modular Subsystem Layout

In accordance with Single Responsibility Principle (SRP) guidelines ([`AGENTS.md`](file:///home/valdis/my-os/AGENTS.md)), the kernel source tree is partitioned into isolated subsystems:

```
wunix/kernel/src/
├── arch/                         # Hardware-specific CPU platform code
│   └── x86_64/
│       ├── cpu.rs                # CR0..CR4 control registers, CPUID, MSR registers
│       ├── gdt.rs                # GDT, TSS and dedicated IST0 exception stack (20 KiB)
│       ├── interrupts.rs         # IDT vector table and CPU exception dispatchers
│       ├── syscall.rs            # Low-level MSR LSTAR entry trampoline (syscall / sysretq)
│       └── timer.rs              # PIT 8254 system timer (100 Hz) and uptime tracker
├── drivers/                      # Device driver subsystem
│   ├── display.rs                # Limine framebuffer output with ANSI escape parser
│   ├── font.rs                   # Monospace 8×16 bitmap font generator (ASCII & Cyrillic)
│   ├── keyboard.rs               # PS/2 keyboard controller, ring buffer, and RU/EN toggle
│   ├── manager.rs                # DeviceManager registry and automated bus scanner
│   ├── registry.rs               # Kernel hardware driver registry
│   ├── rtc.rs                    # CMOS RTC non-volatile real-time clock driver (MC146818)
│   ├── serial.rs                 # COM1 UART 16550 (port 0x3F8) for kernel diagnostics
│   ├── traits.rs                 # Unified device traits: BlockDevice, NetDevice, CharDevice
│   ├── pci/                      # PCI bus subsystem (bus scan, configuration space, BARs)
│   └── virtio/                   # VirtIO virtualization stack (blk, net, gpu, input, pci)
├── fs/                           # Virtual File System (VFS) and I/O abstractions
│   ├── block.rs                  # Block device layer and MBR partition table parser
│   ├── epoll.rs                  # Asynchronous epoll event multiplexer
│   ├── eventfd.rs                # Inter-thread event notification counter (eventfd)
│   ├── fat.rs                    # FAT32 filesystem driver with sector caching
│   ├── fd.rs                     # Process file descriptor table (FD table)
│   ├── handle.rs                 # Uniform FileHandle descriptors (Files, Stdio, Pipes, Sockets)
│   ├── mount.rs                  # Filesystem mount table (mount / umount2)
│   ├── ops.rs                    # Core VFS operations (open, read, write, stat, mkdir, unlink)
│   ├── path.rs                   # C:\ path translation, slash normalization, case-insensitive lookups
│   ├── pipe.rs                   # IPC unidirectional circular ring pipes
│   ├── procfs.rs                 # Diagnostic pseudo-filesystems (/proc, /sys)
│   ├── ramfs.rs                  # Thread-safe in-memory filesystem built upon spin::RwLock
│   ├── socket.rs                 # Network socket file handles
│   ├── tar.rs                    # POSIX ustar initramfs archive parser
│   ├── vfs.rs                    # Core Inode, FileSystem traits, Metadata types, and VfsError
│   └── registry/                 # Windows NT registry subsystem (/win/registry/)
├── memory/                       # Memory management subsystem
│   ├── frame.rs                  # Physical 4 KiB frame allocator parsed from Limine memory map
│   ├── heap.rs                   # Dynamic kernel heap (linked_list_allocator, Box/Vec support)
│   ├── paging.rs                 # 4-level paging translations (PML4, PDP, PD, PT) via HHDM
│   ├── process.rs                # Isolated userland process page tables
│   └── vma.rs                    # Virtual Memory Areas (VMA)
├── net/                          # Network stack subsystem
│   ├── device.rs                 # virtio-net adapter to the smoltcp network stack
│   ├── mod.rs                    # Network initialization, interfaces (eth0, lo)
│   └── socket.rs                 # TCP, UDP, and RAW ICMP socket implementations
├── task/                         # Multitasking, processes, threads, and system calls
│   ├── context.rs                # CPU register context CpuContext and context switching
│   ├── elf.rs                    # ELF64 executable binary loader
│   ├── pe.rs                     # PE32+ (Windows NT) executable binary loader
│   ├── futex.rs                  # Fast Userspace Mutex (futex) synchronization
│   ├── scheduler.rs              # Kernel thread scheduler (preemptive Round-Robin, 50 ms slice)
│   ├── signal.rs                 # POSIX signal delivery (SIGINT, SIGTERM, sigaction)
│   ├── syslib.rs                 # Kernel virtual vDSO library (Syslib) at 0x7FFF_FFFF_0000
│   ├── thread.rs                 # Thread and Process descriptors and lifecycle tracking
│   ├── syscall/                  # Linux / POSIX system call dispatchers (SRP modular design)
│   └── win_syscall/              # Windows NT ABI system call dispatchers (SSN routing)
├── i18n/                         # Internationalization subsystem (MsgKey catalog, tr!, t!)
├── version.rs                    # Operating system version and build metadata
├── lib.rs                        # Kernel root library (kernel_init function)
└── main.rs                       # Kernel entry point (_start) and panic handler
```

---

## 3. Memory Management

Memory management relies upon the Higher-Half Direct Map (HHDM) window provided by the Limine bootloader:

1. **Page Translation (`memory/paging.rs`)**:
   - The kernel configures standard 4-level x86_64 paging: PML4 $\to$ PDPT $\to$ PD $\to$ PT.
   - The HHDM offset allows kernel code to address any physical page directly via: `virt = phys + hhdm_offset`.
2. **Physical Frame Allocator (`memory/frame.rs`)**:
   - Parses the Limine memory map (`memmap_request`), identifies usable RAM regions (`LIMINE_MEMMAP_USABLE`), and allocates 4096-byte physical frames.
3. **Kernel Heap (`memory/heap.rs`)**:
   - Resides at virtual address `0xFFFF_9000_0000_0000`. Starts with a 1 MiB initial capacity.
   - Powered by `linked_list_allocator`, enabling Rust heap collections (`Box`, `Vec`, `String`, `BTreeMap`, and `format!`).
4. **Process Address Spaces (`memory/process.rs`, `vma.rs`)**:
   - Every Ring 3 process receives an independent PML4 page directory.
   - The upper half kernel space is mapped into every process PML4, eliminating the need to flush CR3 during syscall entries.
   - Userland ranges are tracked via Virtual Memory Area (VMA) descriptors with fine-grained access permissions (`PROT_READ`, `PROT_WRITE`, `PROT_EXEC`).

---

## 4. Hardware Bring-Up and CPU Interrupts

1. **GDT and TSS (`arch/x86_64/gdt.rs`)**:
   - Configures 64-bit segment descriptors: kernel code (`0x08`), kernel data (`0x10`), user code (`0x1B` / `0x23`), and user data (`0x20`).
   - Sets up a Task State Segment (TSS) with an emergency 20 KiB IST0 stack. In case of a Double Fault exception, the CPU switches to the IST0 stack automatically, preventing immediate Triple Fault hardware resets.
2. **PIC 8259 Interrupt Controller (`arch/x86_64/interrupts.rs`)**:
   - Remaps legacy BIOS hardware IRQs from vectors 8–15 to 32–47, avoiding collisions with Intel CPU exceptions.
3. **IDT Vector Table (`arch/x86_64/interrupts.rs`)**:
   - Registers typed exception handlers: Divide Error, Breakpoint, Double Fault, General Protection Fault (#GP), and Page Fault (#PF) with detailed CPU register dumps.
4. **PIT 8254 System Timer (`arch/x86_64/timer.rs`)**:
   - Configured for a 100 Hz tick rate (10 ms time slice).
   - Serves system uptime and triggers preemptive multitasking quantum checks.

---

## 5. Processes, Threads, and Task Scheduling

The task subsystem (`task/`) implements a complete Ring 3 multitasking runtime:

1. **Ring 3 Isolation**:
   - Userland code executes with DPL=3 privilege level, preventing access to privileged instructions (`cli`, `sti`, `in`, `out`, CR3 alterations) and protecting kernel memory spaces.
2. **Preemptive Round-Robin Scheduling (`task/scheduler.rs`)**:
   - Distributes CPU execution time with a 50 ms time quantum per thread.
   - Threads can cooperatively yield via `sys_yield` or `NtYieldExecution`.
3. **Context Switching (`task/context.rs`)**:
   - The assembly `context_switch` routine preserves general-purpose registers (`rsp`, `rbp`, `rbx`, `r12`..`r15`, `rflags`) and restores the next scheduled thread state.
4. **Binary Loaders (`task/elf.rs`, `task/pe.rs`)**:
   - `elf.rs`: Parses 64-bit ELF headers, maps `LOAD` segments into process memory, and points the instruction pointer to `e_entry`.
   - `pe.rs`: Loads Portable Executable (PE32+) binaries directly, mapping `.text`, `.data`, and `.rdata` sections.
5. **Thread Local Storage (TLS)**:
   - Configures CPU register `FS_BASE` via `sys_arch_prctl`, supplying required state storage for Rust and C thread runtimes.

---

## 6. System Call Dispatcher (Dual ABI)

Wunix Core implements a high-performance system call gateway using the x86_64 `syscall` instruction:

- The processor register MSR LSTAR is configured with the address of `syscall_entry` ([`arch/x86_64/syscall.rs`](file:///home/valdis/my-os/wunix/kernel/src/arch/x86_64/syscall.rs)).
- MSR SFMASK and STAR registers mask flags and enforce transitions to Ring 0 code selectors.
- **Dual ABI Routing**:
  - Linux/POSIX calls (numbers `0..334`) route into `task/syscall/mod.rs`.
  - Windows NT System Service Numbers (SSN, range `0x0000`..`0x007F`) route into `task/win_syscall/mod.rs`.
- **Pointer Validation**: Pointers originating from userland are verified to ensure they do not exceed `0x0000_7FFF_FFFF_FFFF`, returning `-EFAULT` on violations.

See [System Calls Architecture](syscalls.en.md) for complete details.

---

## 7. Kernel Virtual Library (vDSO / Syslib)

To minimize context-switching overhead for high-frequency operations, a read-only **vDSO / Syslib** page is mapped at `0x0000_7FFF_FFFF_0000` in every user process ([`task/syslib.rs`](file:///home/valdis/my-os/wunix/kernel/src/task/syslib.rs)):

- **Header `SyslibSharedInfo`**: Provides the `WUNIXLIB` magic signature, system timer frequency, boot timestamp, and symbol export tables.
- **Inlined Micro-Functions**: Zero-syscall implementations of `strlen`, `memcpy` (using `rep movsb`), and `memset` (`rep stosb`).
- **Pre-Assembled Fast Trampolines**: Assembly wrappers for `read`, `write`, `getpid`, `close`, and `exit`.

---

## 8. Driver and VirtIO / PCI Subsystems

Device management is coordinated through [`drivers/manager.rs`](file:///home/valdis/my-os/wunix/kernel/src/drivers/manager.rs):

- **PCI Bus (`drivers/pci/`)**: Scans 256 buses, 32 devices, and 8 functions during boot, detecting Vendor/Device IDs and programming Base Address Registers (BARs).
- **VirtIO Virtualization Stack (`drivers/virtio/`)**:
  - `virtio-blk`: Block storage driver handling synchronous and asynchronous 512-byte sector operations.
  - `virtio-net`: Gigabit network adapter driver with TX/RX virtqueues.
  - `virtio-gpu`: Accelerated 2D/3D framebuffer graphics device.
  - `virtio-input`: Absolute pointer and virtual keyboard input devices.

See [Drivers and Device Management](drivers.en.md) for full architectural details.

---

## 9. Network Subsystem

The Wunix network stack integrates `smoltcp` with the hardware `virtio-net` driver:

- **Network Interfaces**: Manages primary interface `eth0` and local loopback `lo`.
- **Supported Protocols**: Complete implementations of Ethernet II, ARP, IPv4, ICMP (Echo), UDP, and windowed TCP.
- **VFS Socket Abstraction**: Sockets are exposed as standard `FileHandle::Socket` descriptors, powering BSD socket syscalls: `socket`, `bind`, `listen`, `accept`, `connect`, `sendto`, `recvfrom`.

See [Network Stack Architecture](network.en.md) for full details.

---

## 10. Framebuffer Console and Localization (i18n)

- **Console Display (`drivers/display.rs`)**: High-resolution Limine linear framebuffer driver supporting hardware-backed scrolling, ANSI escape color sequences, and carriage control characters.
- **Bitmap Font Generator (`drivers/font.rs`)**: Monospace 8×16 font containing ASCII, Extended Latin, and Cyrillic character sets.
- **Kernel Internationalization (`i18n/mod.rs`)**: Kernel messages utilize `tr!` and `t!` macros backed by typed `MsgKey` catalogs. The target language is configured via boot command line (`lang=ru` / `lang=en`) or environment variables.
