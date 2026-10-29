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
| **7** | **First Process & WSH** | Process `/bin/init`, Wunix Shell, core utilities | ✅ **Completed** | Interactive user console in Ring 3, launching binaries from VFS (init, wsh, echo, cat, ls, clear, uname, rm, free, top). |
| **8** | **VirtIO Drivers (QEMU Full Stack)** | PCI Bus Scan, `virtio-blk`, `virtio-net`, `virtio-input`, `virtio-gpu` | 🔄 **In Progress** | Full QEMU virtualization stack: persistent disks, socket networking, virtual mouse and accelerated display. |
| **9** | **Network Stack & Storage Tools** | Linux ABI (`mmap`, FS_BASE, `clone`, `futex`), Go/Rust network stack, sockets, `ping`, `curl`, `fdisk`, `lsblk`, `mount` | ⏳ *Planned* | Networking and storage in VM: packet routing, network/disk CLI tools, static Go/Rust binaries support. |
| **10** | **Redox OS Driver Import** | AHCI SATA, NVMe, Intel e1000/e1000e, Realtek RTL8168/8111, xHCI (USB 3.0) | ⏳ *Planned* | Bare-metal x86_64 support: SSD/NVMe storage controllers, Gigabit Ethernet, and USB stack. |
| **11** | **Driver Audit & Gap Coverage** | Hardware coverage audit, ACPI, APIC/IO-APIC, Wi-Fi, Intel HDA Audio | ⏳ *Planned* | Complete hardware inventory, identification and elimination of driver deficit gaps. |

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
[7. First Process: init / wsh] ─────────────────────────────────► [COMPLETED]
         │
         ▼
[8. VirtIO Drivers: QEMU Full Stack] ───────────────────────────► [IN PROGRESS / NEXT STAGE]
         │
         ▼
[9. Network Stack, Sockets & Disk Tools] (Go/Rust network stack, Linux ABI, ping, curl, fdisk, mount)
         │
         ▼
[10. Redox OS Driver Import] (AHCI SATA, NVMe, Intel e1000, Realtek, xHCI USB)
         │
         ▼
[11. Driver Audit & Gap Coverage] (ACPI, APIC, hardware inventory, Wi-Fi, HDA)
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
- [x] ELF64 executable loader and parser (`kernel/src/task/elf.rs`): `PT_LOAD` segments mapping into Ring 3 PML4, BSS zeroing.
- [x] First user-space init process: `/bin/init` (PID 1) spawning shell and running a supervisor loop.
- [x] Terminal driver (TTY): ANSI SGR color formatting, inverting block cursor, buffered stdin reading.
- [x] Userland system calls: `SYS_EXECVE` (System V ABI argc/argv passing), `SYS_OPEN`, `SYS_CLOSE`, `SYS_UNLINK`, `SYS_GETCWD`, `SYS_CHDIR`, `SYS_SYSINFO`, `SYS_UNAME`, `SYS_GETDENTS64`.
- [x] Basic `wsh` (Wunix Shell): interactive LineEditor with arrow keys, history buffer, prompt (`valdis@wunix:~$ ` / `C:\> `), built-in commands (`cd`, `help`, `history`, `clear`, `exit`), binary execution.
- [x] Porting and building core UNIX utilities in `/bin/`: `init`, `wsh`, `echo`, `cat`, `ls`, `clear`, `uname`, `pwd`, `touch`, `rm`, `top`, `free`.
- [x] Interrupt signal handling: `Ctrl+C` terminates child processes safely without dropping kernel or shell.
- [x] Userland internationalization (`libwunix::i18n`): automatic OS language detection via `/etc/locale` and kernel boot args.

### Phase 8: VirtIO Drivers (QEMU Full Stack Virtualization)
- [ ] PCI / PCIe bus scan (`pci::scan()`, configuration space enumeration, Vendor ID, Device ID, BARs).
- [ ] VirtIO queue infrastructure (Virtqueue: Split Virtqueue descriptors, Available and Used rings).
- [ ] Block storage driver `virtio-blk` (LBA sector read/write, persistent disk storage in QEMU).
- [ ] Network adapter driver `virtio-net` (Ethernet frame RX/TX, integration with network stack).
- [ ] Input device driver `virtio-input` (tablet/mouse with absolute cursor coordinate support).
- [ ] Video display driver `virtio-gpu` (hardware display mode switching, 2D acceleration).

### Phase 9: Network Stack, Sockets & Base Image Expansion
- [ ] Minimal Linux ABI in kernel for static binaries (Go / Rust):
  - [ ] Memory allocation syscalls `SYS_MMAP` (9) and `SYS_MUNMAP` (11).
  - [ ] Architecture control syscall `SYS_ARCH_PRCTL` (158) to set `FS_BASE` register (Go goroutine `g` context & thread-local storage).
  - [ ] Multithreading and synchronization syscalls: `SYS_CLONE` (56), `SYS_FUTEX` (202), `SYS_SET_TID_ADDRESS` (218).
  - [ ] Entropy and signals: `SYS_GETRANDOM` (318), stub handlers for `SYS_RT_SIGACTION` (13) and `SYS_RT_SIGPROCMASK` (14).
- [ ] Network stack and BSD socket interface:
  - [ ] Packet processing for `virtio-net` driver (via `smoltcp` in Rust or userland network daemon in Go).
  - [ ] Socket syscalls: `SYS_SOCKET` (41), `SYS_CONNECT` (42), `SYS_SENDTO` (44), `SYS_RECVFROM` (45), `SYS_BIND` (49).
  - [ ] Protocol support: ARP, IPv4, ICMP (Echo Request/Reply for ping), UDP (DNS resolution), TCP.
- [ ] Network utility suite in base image:
  - [ ] `ping` utility: ICMP Echo requests and RTT latency measurement.
  - [ ] `curl` / `wget` utility: HTTP GET requests and file downloads.
  - [ ] `ip` utility: interface inspection, IP address and default gateway route assignment.
  - [ ] `ss` / `netstat` utility: active socket and connection monitoring.
- [ ] Storage utilities and filesystem support:
  - [ ] `lsblk` utility: block device enumeration (`/dev/vda`).
  - [ ] `fdisk` utility: MBR / GPT partition table editing on persistent disk.
  - [ ] `mount` and `umount` utilities: mounting partition filesystems (FAT32 / ext2) into VFS.
  - [ ] Persistence verification: saving user files to QCOW2 virtual disk across QEMU reboots.

### Phase 10: Redox OS Driver Import & Adaptation
- [ ] Adapter layer for Redox OS bus and device traits mapped to Wunix VFS and DeviceManager.
- [ ] AHCI SATA storage controller driver (support for physical HDDs and SATA SSDs).
- [ ] NVMe controller driver (fast PCIe M.2 SSDs, Submission/Completion ring queues).
- [ ] Intel Gigabit Ethernet: `e1000` and `e1000e` NIC drivers.
- [ ] Realtek Gigabit Ethernet: `rtl8168` and `rtl8111` onboard NIC drivers.
- [ ] USB xHCI Host Controller (USB 3.0: support for flash drives, USB keyboards and mice).

### Phase 11: Hardware Inventory & Driver Deficit Coverage
- [ ] Comprehensive hardware coverage audit: supported controllers inventory and deficit gap mapping.
- [ ] ACPI subsystem (RSDP, MADT, FADT, DSDT table parsing for power management and SMP cores).
- [ ] APIC / IO-APIC interrupt controller (replacing legacy PIC 8259 for modern multicore CPUs).
- [ ] Audio subsystem: Intel High Definition Audio (HDA / Realtek ALC) sound controller driver.
- [ ] Wireless networking: basic Wi-Fi stack (Intel Wireless WiFi / Atheros).
- [ ] Hardware diagnostic utilities for Wunix Shell: `lspci`, `lsusb`, `lshw`.
