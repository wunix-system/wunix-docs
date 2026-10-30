[ 🇷🇺 Русский ](README.ru.md) | [ 🇬🇧 English ](README.en.md)

# Wunix Operating System Documentation

This directory contains the official technical documentation of **Wunix**, an independent 64-bit UNIX-like operating system written from scratch in Rust.

---

## Documentation Structure

The documentation is organized into technical domains, with every document maintained in both Russian and English using a consistent engineering style:

### 1. Architecture and System Design ([`architecture/`](architecture/))
- **[System Architecture Overview](architecture/overview.en.md)** ([Русский](architecture/overview.ru.md)):
  - Project vision, architectural layers (Applications, Desktop, Base, System API, Core, Hardware), WPM/WSP package managers, and release model.
- **[Kernel Architecture](architecture/kernel.en.md)** ([Русский](architecture/kernel.ru.md)):
  - 64-bit Long Mode kernel, Higher-Half layout, Limine boot protocol, Ring 3 isolation, Round-Robin preemptive scheduler (50 ms slice), and modular SRP codebase hierarchy.
- **[Virtual File System (VFS) and Hybrid Path Architecture](architecture/vfs.en.md)** ([Русский](architecture/vfs.ru.md)):
  - Unified VFS root, transparent `C:\` Windows drive projection, uniform `FileHandle` descriptors, in-memory RamFS, `BlockDevice` / MBR partitions, FAT32 driver, ProcFS, and Windows NT virtual registry.
- **[System Calls and Dual ABI Architecture](architecture/syscalls.en.md)** ([Русский](architecture/syscalls.ru.md)):
  - High-performance MSR LSTAR `syscall`/`sysret` gateway, specifications for all 60+ Linux/POSIX system calls, and 24 Windows NT System Service Numbers (SSN).
- **[System Libraries and Runtime](architecture/libraries.en.md)** ([Русский](architecture/libraries.ru.md)):
  - `libwunix` userland base library, vDSO / Syslib page (`0x7FFF_FFFF_0000`), Kernel Crates, and planned SDK roadmap (`libwui`, `libc`, Rust `std`).
- **[Device Drivers and Hardware Subsystems](architecture/drivers.en.md)** ([Русский](architecture/drivers.ru.md)):
  - `DeviceManager`, automated PCI bus scanning, VirtIO virtualization stack (`blk`, `net`, `gpu`, `input`), UART 16550 COM1, PS/2 keyboard with RU/EN toggling, and CMOS RTC.
- **[Network Stack Architecture](architecture/network.en.md)** ([Русский](architecture/network.ru.md)):
  - Integration of `smoltcp` with `virtio-net`, BSD socket handles, Ethernet/ARP/IPv4/ICMP/UDP/TCP protocols, and CLI utilities (`ping`, `curl`, `ip`, `ss`).

### 2. Manual Pages and Command Reference ([`man/`](man/))
- **[Manual Pages Guide and Command Catalog](man/README.en.md)** ([Русский](man/README.ru.md)):
  - Guide to using `man`, comprehensive reference covering Section 1 user utilities (`cat`, `ls`, `curl`, `wsh`, `top`...) and Section 8 administration tools (`fdisk`, `mount`, `ip`, `ping`, `init`...).

### 3. Localization Subsystem ([`localization/`](localization/))
- **[Localization Subsystem Architecture](localization/README.en.md)** ([Русский](localization/README.ru.md)):
  - Internationalization design, atomic language switching, `cmdline` boot flags (`lang=ru`/`lang=en`), 8×16 bitmap font, and keyboard layout translation.
- **[Localization and Translation Guide](localization/translation-guide.en.md)** ([Русский](localization/translation-guide.ru.md)):
  - Adding new language catalogs, exhaustive `MsgKey` mapping, and `tr!` / `t!` macro usage.

### 4. Development Roadmap ([`roadmap/`](roadmap/))
- **[Development Roadmap and Status](roadmap/roadmap.en.md)** ([Русский](roadmap/roadmap.ru.md)):
  - Milestone summary, status of completed phases 1–9, active phase 10 (Multi-User & Security), and planned phases 11–12 (Redox OS driver import and hardware coverage audit).

### 5. Developer Guides ([`guides/`](guides/))
- **[Building, Booting, and Debugging Wunix](guides/development.en.md)** ([Русский](guides/development.ru.md)):
  - Host prerequisites, automated execution via `tools/run.sh`, userland compilation, FAT32 disk image generation, QEMU configuration with VirtIO devices, and GDB/COM1 debugging.
