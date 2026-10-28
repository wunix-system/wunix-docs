[ 🇷🇺 Русский ](roadmap.ru.md) | [ 🇬🇧 English ](roadmap.en.md)

# Wunix Development Roadmap and Status

This document tracks the milestones of the Wunix kernel and base system, current progress, and the detailed task list.

---

## 1. Summary Table of Development Phases

| # | Development Phase | Key Components | Status | Result / Completion Criteria |
|:---:|:---|:---|:---:|:---|
| **1** | **Hardware Bring-up & CPU** | GDT, TSS, IST0, IDT, PIC 8259, Serial COM1, Limine, i18n | ✅ **Completed** | Kernel boots reliably in 64-bit Long Mode, handles CPU interrupts, supports i18n. |
| **2** | **Memory Manager** | Frame Allocator, Paging (HHDM), Heap (1 MiB), `alloc` | ✅ **Completed** | Dynamic memory (`Box`, `Vec`, `String`), heap in Higher-Half (`0xFFFF_9000_...`), UNIX autoconfig log. |
| **3** | **Basic I/O & Timers** | PS/2 Keyboard (Ring Buffer, Alt+Shift), PIT 8254 Timer | ✅ **Completed** | Thread-safe input queue, system uptime counter, `Ctrl+C` interrupt handling. |
| **4** | **Kernel Multitasking** | TCB, thread stacks, `context_switch`, scheduler | ✅ **Completed** | Preemptive multitasking (Round-Robin, 50 ms timeslice), kernel thread stack isolation. |
| **5** | **VFS & Storage** | VFS traits, Ramdisk / Initramfs (TAR), DeviceManager | ✅ **Completed** | Root FS (`/`) mounted, initramfs extracted, hybrid drive `C:\`, file read/write validation. |
| **6** | **Userland (Ring 3)** | Process address space, `syscall` / `sysret`, Cargo Wksp | ✅ **Completed** | Ring 0/3 privilege separation, isolated PML4, fast syscalls, `libwunix` crate, #GP isolation. |
| **7** | **First Process & WSH** | Process `/bin/init`, Wunix Shell, core utilities | 🔄 **Next Phase** | Interactive user console in Ring 3, launching binaries from VFS. |

---

## 2. Architectural Pipeline

```
[1. Hardware Bring-up] (GDT, IDT, PIC, Framebuffer, Serial, Limine) ──► [COMPLETED]
         │
         ▼
[2. Memory Manager] (Higher-Half, Paging, 1MiB Heap, alloc) ─────────► [COMPLETED]
         │
         ▼
[3. Basic I/O & Timers] (PS/2 buffer, Alt+Shift, PIT 100 Hz) ────────► [COMPLETED]
         │
         ▼
[4. Multitasking] (Kernel threads, context switching) ───────────────► [COMPLETED]
         │
         ▼
[5. File System & VFS] (Ramdisk / initramfs loading) ────────────────► [COMPLETED]
         │
         ▼
[6. Transition to Ring 3] (User Space isolation, syscall / sysret) ──► [COMPLETED]
         │
         ▼
[7. First Process: init / wsh] ────► [IN PROGRESS / NEXT STAGE]
```

---

## 3. Detailed Task Status

### Phase 1: Hardware Bring-up & Boot
- [x] GDT descriptors and TSS (Task State Segment).
- [x] Dedicated IST0 emergency stack (20 KB) for Double Fault exception.
- [x] IDT (Interrupt Descriptor Table) for Breakpoint, Double Fault, Page Fault.
- [x] PIC 8259 interrupt controller remapped to vectors `32..47`.
- [x] UART 16550 serial port driver (COM1, port `0x3F8`) for terminal debugging.
- [x] Modern **Limine v0.6.5** bootloader (Higher-Half, UEFI & BIOS).
- [x] Modular kernel structure adhering to Section 5 of AGENTS.md.
- [x] Kernel internationalization (i18n) subsystem with RU/EN boot support.

### Phase 2: Memory Subsystem
- [x] Limine HHDM (Higher-Half Direct Map) integration.
- [x] 4-level page table translator via `OffsetPageTable`.
- [x] Physical frame allocator `LimineFrameAllocator` (4 KiB frames from `MEMMAP_USABLE`).
- [x] Lightweight heap allocator `linked_list_allocator` with `extern crate alloc`.
- [x] Heap relocated to Higher-Half (`0xFFFF_9000_0000_0000`, 1 MiB).
- [x] `MemoryStats` structure computing real and available RAM (`real mem`, `avail mem`).
- [x] Strict UNIX/BSD autoconfiguration logging style.

### Phase 3: Basic I/O & Timers
- [x] PS/2 keyboard driver with circular event buffer (`RingBuffer<char, 128>`).
- [x] Support for modifier keys: Shift, Ctrl, Alt, CapsLock.
- [x] Dual keyboard layout (EN / RU) with `Alt+Shift` toggle.
- [x] `Ctrl+C` interrupt handling to cancel the current input line.
- [x] PIT 8254 system timer initialization (100 Hz, divisor 11932).
- [x] Global atomic tick counter and `uptime_ms` system time function.

### Phase 4: Kernel Multitasking & Scheduler
- [x] Thread Control Block (TCB) structure.
- [x] Individual kernel thread stacks (32 KiB).
- [x] Assembly CPU register save/restore procedure (`context_switch`).
- [x] Cooperative thread switching (`yield_now`).
- [x] Preemptive Round-Robin scheduler (50 ms timeslice) triggered by PIT timer ticks.
- [x] Clean deallocation of terminated thread stacks (`garbage` in `Scheduler`).

### Phase 5: Virtual File System (VFS) & Storage
- [x] VFS abstraction traits: `Inode`, `FileSystem`, `Metadata`, `DirEntry`, `VfsError`.
- [x] Thread-safe in-memory file system `RamFs` (`RamInode`).
- [x] Windows NT path compatibility (`C:\...`) and case-insensitive resolution.
- [x] POSIX ustar TAR archive unpacker (`unpack_tar`) extracting directly into VFS inodes.
- [x] Loading initramfs archive via Limine protocol.
- [x] Hybrid directory skeleton (`/bin`, `/etc`, `C:\Windows`, `/home/user`).

### Phase 6: Userland (Ring 3) & System Calls
- [x] Cargo Workspace restructuring (`kernel`, `libs/libwunix`).
- [x] GDT descriptors for Ring 3 (User Code 0x20, User Data 0x18) and kernel `rsp0` stack in TSS (32 KiB).
- [x] MSR register configuration for x86_64 `syscall` / `sysret` (`IA32_STAR`, `IA32_LSTAR`, `IA32_FMASK`, `EFER.SCE`).
- [x] Assembly entry point `syscall_entry` with stack switching, context preservation, and System V calling convention.
- [x] System call dispatcher (`sys_write`, `sys_read`, `sys_exit`, `sys_yield`) with user pointer validation.
- [x] Isolated process address spaces (`UserAddressSpace`: separate PML4, zeroed lower half, shared kernel Higher-Half).
- [x] User/Supervisor page protection (`PageTableFlags::USER_ACCESSIBLE`) for user code and stack.
- [x] Fault isolation for user-mode `#GP` and `#PF` exceptions with graceful process termination.
- [x] User-space no_std system call wrapper library `libwunix` (`wunix/libs/libwunix`).
- [x] Successful Ring 3 verification: `sys_write` / `sys_exit` execution and privileged instruction `#GP` trap.

### Phase 7: Userland Environment & Wunix Shell
- [ ] ELF64 executable loader and parser to run binaries from VFS.
- [ ] First user-space init process: `/bin/init`.
- [ ] Terminal driver (TTY) for console I/O in Ring 3.
- [ ] Basic `wsh` (Wunix Shell): prompt, argument parsing, running commands.
- [ ] Porting core UNIX utilities: `echo`, `cat`, `ls`, `clear`, `uname`.
