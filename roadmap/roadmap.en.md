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
| **8** | **VirtIO Drivers (QEMU Full Stack)** | PCI Bus Scan, `virtio-blk`, `virtio-net`, `virtio-input`, `virtio-gpu` | ✅ **Completed** | Full QEMU virtualization stack: persistent disks, socket networking, virtual mouse and accelerated display. |
| **9** | **Network Stack & Storage Tools** | Linux ABI (`mmap`, FS_BASE, `clone`, `futex`), Rust network stack (`smoltcp`), BSD sockets, `ping`, `curl`, `ip`, `fdisk`, `lsblk`, `mount` | ✅ **Completed** | Production Linux ABI (Ring 3 runtimes verified). Rust network stack: packet RX/TX, TCP/UDP/ICMP, FAT32 persistent disk mounting, persistence. |
| **10** | **Users & Access Control** | UID/GID model, `/etc/passwd`, `/etc/shadow`, POSIX VFS permissions (rwx, suid), `login`, `su`, `sudo`, `sudoadd`, `chmod`, `chown` | ✅ **Completed** | Secure multi-user environment, password authentication, process privilege separation, and VFS access control. |
| **11** | **Wunix Bootloader & TUI Installer** | Boot Menu (Live-ISO / Install), TUI Installer (`winstall`), MBR/GPT partitioning, disk deployment | ⏳ *Planned* | Autonomous Wunix booting from disk/USB, Live-ISO / Install mode selection, terminal UI disk installation. |
| **12** | **Virtualization Hypervisor Drivers** | VirtualBox (VBoxGuest, VBoxSVGA), VMware (SVGA II, vmxnet3, PVSCSI), Hyper-V (VMBus, NetVSC, StorVSC), Proxmox/Xen | ⏳ *Planned* | Seamless execution across all hypervisors: VirtualBox, VMware Workstation/ESXi, Microsoft Hyper-V, Proxmox VE, KVM, and Xen. |
| **13** | **Full Redox OS Driver Import** | Complete Redox driver adaptation: AHCI SATA, NVMe, IDE, e1000/e1000e/igb, Realtek 8168/8139, xHCI/EHCI/UHCI USB, HDA/AC'97 | ⏳ *Planned* | Comprehensive physical x86_64 controller support: SATA/NVMe disks, Gigabit Ethernet, USB 1.1–3.0 stack, and audio. |
| **14** | **Hardware Audit & Gap Coverage** | Hardware coverage audit, ACPI (DSDT/MADT/AML), Local APIC / IO-APIC, SMP, Wi-Fi stack, Intel HDA | ⏳ *Planned* | Exhaustive hardware inventory, SMP multicore bring-up, modern interrupt routing, power management, and Wi-Fi. |
| **15** | **Advanced File Systems (EXT4, Btrfs)** | EXT2/EXT3/EXT4 driver (extents, inodes, JBD2), Btrfs (B-Tree, CoW, subvolumes, snapshots, CRC32c), `mkfs`, `fsck` | ⏳ *Planned* | Industrial-grade journaling and CoW filesystems: EXT4/Btrfs partition creation, instant snapshots, data integrity. |

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
[8. VirtIO Drivers: QEMU Full Stack] ───────────────────────────► [COMPLETED]
         │
         ▼
[9. Rust Network Stack & Storage Tools] ────────────────────────► [COMPLETED]
         │
         ▼
[10. Users & Access Control] (UID/GID, /etc/passwd, shadow, VFS perms, su/sudo) ──────────► [COMPLETED]
         │
         ▼
[11. Wunix Bootloader & TUI Installer] (Live-ISO vs Install, TUI installer, disk deploy)
         │
         ▼
[12. Hypervisor Virtualization Drivers] (VirtualBox, VMware, Hyper-V, Proxmox/KVM, Xen)
         │
         ▼
[13. Complete Redox OS Driver Import] (AHCI SATA, NVMe, e1000/igb, Realtek, xHCI/EHCI USB, HDA)
         │
         ▼
[14. Hardware Audit & Gap Coverage] (ACPI, APIC/IO-APIC, SMP multicore, Wi-Fi stack)
         │
         ▼
[15. Advanced File Systems] (EXT4 extents/journaling, Btrfs CoW/snapshots/B-Tree)
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
- [x] PCI / PCIe bus scan (`pci::scan()`, configuration space enumeration, Vendor ID, Device ID, BARs).
- [x] VirtIO queue infrastructure (Virtqueue: Split Virtqueue descriptors, Available and Used rings).
- [x] Block storage driver `virtio-blk` (LBA sector read/write, persistent disk storage in QEMU).
- [x] Network adapter driver `virtio-net` (Ethernet frame RX/TX, integration with network stack).
- [x] Input device driver `virtio-input` (tablet/mouse with absolute cursor coordinate support).
- [x] Video display driver `virtio-gpu` (hardware display mode switching, 2D acceleration).

### Phase 9: Rust Network Stack & Storage Tools
- [x] Production Linux ABI in kernel for static binaries (Rust / C / Go):
  - [x] Memory allocation syscalls `SYS_MMAP` (9) and `SYS_MUNMAP` (11) with VMA tracker and demand paging.
  - [x] Architecture control syscall `SYS_ARCH_PRCTL` (158) to set `FS_BASE` register (thread TLS) with context switch preservation.
  - [x] Multithreading and synchronization syscalls: `SYS_CLONE` (56), `SYS_FUTEX` (202), `SYS_SET_TID_ADDRESS` (218).
  - [x] Entropy and signals: `SYS_GETRANDOM` (318), `SYS_RT_SIGACTION` (13), `SYS_RT_SIGPROCMASK` (14), `SYS_SIGALTSTACK` (131).
  - [x] Asynchronous I/O: epoll reactor (`SYS_EPOLL_CREATE1`, `SYS_EPOLL_CTL`, `SYS_EPOLL_WAIT`), ring buffer pipes (`SYS_PIPE2`), `eventfd2`, `SYS_FCNTL` (O_NONBLOCK).
  - [x] System V ABI stack with Auxiliary Vectors (`auxv`), `envp` and 16-byte alignment.
  - [x] Complete runtime syscall verification: successfully passed comprehensive tests for concurrency/channels, allocator/GC, and async I/O/timers.
- [x] Kernel network stack in Rust (`smoltcp`) and BSD socket interface:
  - [x] Packet processing for `virtio-net` driver integrated with `smoltcp` (in-kernel Ethernet frame handling, ring RX/TX buffers).
  - [x] Linux socket syscalls: `SYS_SOCKET` (41), `SYS_CONNECT` (42), `SYS_SENDTO` (44), `SYS_RECVFROM` (45), `SYS_BIND` (49), `SYS_LISTEN` (50), `SYS_ACCEPT` (43).
  - [x] Protocol support: ARP, IPv4, ICMP (Echo Request/Reply for ping), UDP (DNS resolution), and TCP (reliable stream connection).
- [x] Network utility suite in base image (Userland in Rust):
  - [x] `ping` utility: ICMP Echo requests and RTT latency measurement.
  - [x] `curl` / `wget` utility: HTTP GET requests and file downloads.
  - [x] `ip` utility: interface inspection, IP address and default gateway route assignment.
  - [x] `ss` / `netstat` utility: active socket and connection monitoring.
- [x] Storage utilities and filesystem support:
  - [x] Integration of `virtio-blk` driver with kernel device subsystem (`/dev/vda`).
  - [x] `lsblk` utility: block device enumeration.
  - [x] `fdisk` utility: MBR / GPT partition table editing on persistent disk.
  - [x] Persistent filesystem support (FAT32 / ext2) and syscalls `SYS_MOUNT` (165), `SYS_UMOUNT2` (166).
  - [x] `mount` and `umount` utilities: mounting partition filesystems into VFS (`/mnt/...`).
  - [x] Persistence verification: saving user files to QCOW2 virtual disk across QEMU reboots.
- [x] Hardware diagnostics and man page utilities:
  - [x] Utilities `lspci`, `lshw`, `lsdev`, `lsmod`, `modinfo`, `hwclock`, `fbset`.
  - [x] `man` utility with offline documentation manual pages (`/usr/share/man/ru/man1/`, `/usr/share/man/en/man1/`).

### Phase 10: Users & Access Control (Multi-User & Security)
- [x] Process and user identity model:
  - [x] Account fields in TCB/PCB: UID, GID, Effective UID (EUID), Effective GID (EGID), Saved UID (SUID), Saved GID (SGID), `umask`.
  - [x] Privileged superuser `root` (UID 0, GID 0) with default initial session in `/root` without password.
  - [x] Identity syscalls: `SYS_GETUID` (102), `SYS_SETUID` (105), `SYS_GETGID` (104), `SYS_SETGID` (106), `SYS_GETEUID` (107), `SYS_GETEGID` (108), `SYS_UMASK` (95).
- [x] Account configuration databases in VFS:
  - [x] File `/etc/passwd`: accounts, UID/GID, home directories (`/home/<user>`), login shell (`/bin/wsh`).
  - [x] File `/etc/group`: user groups (`root`, `wheel`) and membership lists.
  - [x] File `/etc/shadow`: secure storage of hashed passwords (FIPS 180-4 SHA-256 with salt in `$5$salt$hash` format) restricted to `root` (`0600`).
- [x] VFS access permissions (POSIX Permissions):
  - [x] Inode metadata: permission mode `mode` (Owner, Group, Others with `rwx` flags), `SUID` (04000), `SGID` (02000), `Sticky bit` (01000) for `/tmp`.
  - [x] Kernel-level permission checks during `SYS_OPEN`, `SYS_READ`, `SYS_WRITE`, `SYS_EXECVE`, `SYS_UNLINK`, `SYS_MKDIR`, `SYS_RMDIR`.
  - [x] File creation mask `umask` (`SYS_UMASK`, 95) and permission inheritance in RamFS/Initramfs.
  - [x] Permission management syscalls: `SYS_CHMOD` (90), `SYS_FCHMOD` (91), `SYS_CHOWN` (92), `SYS_FCHOWN` (93).
- [x] User and session management utilities for WSH:
  - [x] `login` utility: terminal password authentication (passwordless root login, SHA-256 hash verification for users, `*` masking, backspace handling).
  - [x] `su` and `sudo` utilities: privilege switching and executing commands as root (`wheel` group members).
  - [x] `sudoadd` utility: granting and revoking administrative sudo privileges (`wheel` group membership).
  - [x] `id` and `whoami` utilities: displaying current user identity and groups with name resolution.
  - [x] `passwd` utility: interactive account password modification with `*` masking and SHA-256 salt/hash generation.
  - [x] `useradd` and `userdel` utilities: user account creation and deletion with `/home/<user>` directory lifecycle.
  - [x] `chmod` and `chown` utilities: modifying file permissions (octal and symbolic modes) and ownership.
  - [x] `wsh` and `init` integration: starting initial session as root in `/root`, `#` prompt for root and `>` for unprivileged users, automated respawn of `/bin/login` on session exit (`exit`).

### Phase 11: Wunix Bootloader & TUI Installer (Bootloader, Live-ISO & Installer)
- [ ] Wunix Bootloader architecture and multi-entry menu:
  - [ ] Configuration module and boot menu supporting both BIOS and UEFI (Higher-Half x86_64).
  - [ ] `Live-ISO` Mode ("Try Wunix without installing"): complete system bring-up in RamFS, running without touching local disks.
  - [ ] `Normal Boot` Mode: standard boot sequence of an installed system from local block storage (`/dev/vda` / `/dev/sda` / NVMe).
  - [ ] `Safe / Verbose Mode`: boot with forced VGA text mode and verbose dmesg kernel logging to COM1 and screen.
  - [ ] `Installer` Mode: direct execution of the TUI system installer upon boot.
- [ ] Terminal UI disk installer (`/sbin/winstall` or `/bin/installer`):
  - [ ] TUI interface built on ANSI/VT100 escape sequences (dialog boxes, arrow-navigated menus, text input fields, progress indicators).
  - [ ] Automatic detection of connected block storage devices via VFS (`/dev/vda`, `/dev/sda`, `/dev/nvme0n1`).
  - [ ] Partitioning wizard: automatic layout (ESP/Boot and root `/` partitions) or manual partitioning via built-in `fdisk`.
  - [ ] Target partition formatting (FAT32 / upcoming EXT4) and directory structure initialization.
  - [ ] Target root partition mounting to `/mnt` and file-by-file deployment of Wunix Base (kernel, initramfs, `/bin`, `/sbin`, `/etc`, `/usr`).
  - [ ] Boot sector / Stage 2 loader deployment to target disk for standalone booting without installation media.
  - [ ] Initial system setup wizard: hostname configuration (`/etc/hostname`), root password configuration, and optional unprivileged user creation.

---

### Phase 12: Virtualization Hypervisor Drivers (VirtualBox, VMware, Hyper-V, Proxmox/KVM, Xen)
- [ ] VirtualBox (Oracle VM VirtualBox):
  - [ ] VirtualBox Guest PCI Device driver (Vendor `0x80EE`, Device `0xCAFE`).
  - [ ] VirtualBox Graphics Adapter driver (VBoxVGA / VBoxSVGA, Vendor `0x80EE`, Device `0xBEEF`).
  - [ ] VirtualBox Mouse Integration (absolute pointer coordinates without cursor capture).
  - [ ] Host shared folders client VirtualBox Shared Folders (VBoxSF) mounted into VFS.
  - [ ] Host time and event guest synchronization service.
- [ ] VMware (Workstation, Player, ESXi):
  - [ ] VMware Backdoor I/O channel (ports `0x5658`/`0x5659`, signature `0x564D5868`) for hypervisor detection and telemetry.
  - [ ] VMware SVGA II display controller (Vendor `0x15AD`, Device `0x0405`: accelerated 2D/3D operations, hardware cursor).
  - [ ] VMware vmxnet3 virtual NIC (Vendor `0x15AD`, Device `0x07B0`: high-performance multiqueue paravirtual network adapter).
  - [ ] VMware PVSCSI disk controller (Vendor `0x15AD`, Device `0x07C0`: paravirtual SCSI host adapter).
  - [ ] VMware Absolute Mouse (VMMOUSE) integration.
- [ ] Microsoft Hyper-V (Windows Server, Azure, Windows 10/11 Pro):
  - [ ] Microsoft VMBus synthetic device architecture (message-passing channels, GPADL ring buffers).
  - [ ] Hyper-V Synthetic Video adapter.
  - [ ] NetVSC network adapter (Hyper-V Network Virtual Service Client).
  - [ ] StorVSC disk controller (Hyper-V Storage Virtual Service Client).
  - [ ] Core Hyper-V Integration Services: Heartbeat, Shutdown, and KVP (Key-Value Pair) exchange.
- [ ] Proxmox VE, KVM / QEMU, and Xen:
  - [ ] VirtIO extensions for Proxmox/KVM: VirtIO Balloon (dynamic memory management), VirtIO Console, VirtIO RNG, VirtIO SCSI.
  - [ ] Xen Paravirtualization interface: Hypercalls, Grant Tables, and Xen PV block and network drivers.

---

### Phase 13: Full Redox OS Driver Import & Adaptation
- [ ] Redox OS driver compatibility layer (Redox Driver Abstraction Layer):
  - [ ] Redox bus and device traits (PCI, MMIO, DMA buffers) mapped to Wunix DeviceManager.
- [ ] Storage controllers and block devices:
  - [ ] AHCI SATA controller driver (Serial ATA AHCI 1.0..1.3.1, NCQ command queues, FIS structures, SSD and HDD support).
  - [ ] NVMe controller driver (PCIe M.2 / U.2, NVM Express 1.x..2.x, Submission/Completion queue rings, 4K sector support).
  - [ ] Legacy IDE / PATA controller driver (ATA/ATAPI, Primary/Secondary channels, PIO and DMA modes).
- [ ] Ethernet Network Adapters (NICs):
  - [ ] Intel Gigabit Ethernet: `e1000` (82540EP, 82545EM), `e1000e` (82573, 82574L, I217/I218/I219), and `igb` (82575, 82576, I350).
  - [ ] Realtek Ethernet: `rtl8168` / `rtl8111` (PCIe Gigabit) and `rtl8139` (Fast Ethernet 10/100).
- [ ] USB Host Controllers & USB Stack:
  - [ ] xHCI controller driver (eXtensible Host Controller Interface, USB 3.0/3.1/3.2, TRB rings, device and slot contexts).
  - [ ] EHCI controller driver (Enhanced Host Controller Interface, USB 2.0 High-Speed).
  - [ ] UHCI / OHCI controller drivers (USB 1.1 Full/Low-Speed).
  - [ ] USB device stack: USB Mass Storage (flash drives, card readers) and USB HID (keyboards, mice).
- [ ] Audio drivers ported from Redox OS:
  - [ ] Intel High Definition Audio driver (Intel HDA / Realtek ALC codecs).
  - [ ] AC'97 Audio controller driver.
  - [ ] Sound Blaster 16 driver (QEMU emulation and vintage hardware).

---

### Phase 14: Hardware Coverage Audit & Driver Gap Elimination
- [ ] Comprehensive hardware coverage audit: supported controllers inventory and deficit gap mapping.
- [ ] ACPI (Advanced Configuration and Power Interface) subsystem:
  - [ ] RSDP, RSDT, and XSDT root table parsing.
  - [ ] MADT/APIC table parsing for multicore interrupt routing.
  - [ ] FADT and DSDT parsing: basic AML (ACPI Machine Language) bytecode interpreter for sleep, wake, and poweroff.
- [ ] APIC / IO-APIC Interrupt Controllers & SMP (Symmetric Multiprocessing):
  - [ ] Local APIC (per-CPU core) and motherboard IO-APIC initialization.
  - [ ] MSI and MSI-X (Message Signaled Interrupts) queue support for PCIe devices.
  - [ ] Inter-Processor Interrupts (IPI) and secondary CPU core initialization (SMP Bring-up, SIPI sequence).
- [ ] Wireless networking (Wi-Fi stack):
  - [ ] Core IEEE 802.11 wireless network stack architecture (frame scanning, Beacon, Probe, association).
  - [ ] Drivers for common Wi-Fi chipsets (Intel Wireless-N / Wireless-AC, Atheros ath9k/ath10k).
- [ ] Hardware diagnostic utilities for Wunix Shell: `lspci`, `lsusb`, `lshw`, `lsdev`.

---

### Phase 15: Advanced File Systems (EXT4, Btrfs)
- [ ] EXT filesystem family (EXT2 / EXT3 / EXT4):
  - [ ] EXT4 superblock and Block Group Descriptors parsing.
  - [ ] EXT4 inode table (256-byte extended inodes, nanosecond timestamps).
  - [ ] Extent Tree support (`ext4_extent_header`, `ext4_extent_idx`, `ext4_extent`) for efficient large file addressing (> 2 TiB).
  - [ ] HTree directory indexing (Half-MD4 / TEA directory hash indexing).
  - [ ] JBD2 journaling integration (reading, consistency checking, transaction replay after dirty unmounts).
  - [ ] Userland utilities: `mkfs.ext4` formatting and `fsck.ext4` integrity verification.
- [ ] Next-generation Btrfs filesystem (B-Tree Filesystem):
  - [ ] Copy-on-Write (CoW) architecture and B-Trees (Root Tree, Chunk Tree, FS Tree).
  - [ ] Subvolume and snapshot management.
  - [ ] Data and metadata checksumming (CRC32c / xxHash / SHA-256) for silent data corruption (bit rot) detection.
  - [ ] Transparent block-level compression (Zstandard / LZO).
  - [ ] Userland utility `btrfs` (`btrfs subvolume create`, `btrfs subvolume list`, `btrfs subvolume snapshot`).
- [ ] VFS and system utility integration:
  - [ ] Support for mounting EXT4 and Btrfs partitions as root `/` or data directories (`/home`, `/var`).
  - [ ] Extended attributes (xattr) and POSIX ACL permissions support.
