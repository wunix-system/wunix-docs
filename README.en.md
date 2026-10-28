[ 🇷🇺 Русский ](README.ru.md) | [ 🇬🇧 English ](README.en.md)

# Wunix Operating System Documentation

This repository contains official technical documentation for the **Wunix** operating system, an independent UNIX-like platform developed in Rust.

---

## Documentation Structure

The documentation is organized into modular topic sections. Every document is available in both Russian and English:

### 1. Architecture and System Design ([`architecture/`](architecture/))
- **Wunix System Overview**:
  - [🇷🇺 Русский](architecture/overview.ru.md) | [🇬🇧 English](architecture/overview.en.md)
  - System concept, architectural layers (Applications, Desktop, Base, System API, Core, Hardware), package management model (WPM/WSP), and release lifecycles.
- **Wunix Core Kernel Architecture**:
  - [🇷🇺 Русский](architecture/kernel.ru.md) | [🇬🇧 English](architecture/kernel.en.md)
  - Kernel internals: Higher-Half memory model, Limine boot protocol, paging and physical frame management, interrupts, and I/O drivers.

### 2. Localization Subsystem ([`localization/`](localization/))
- **Localization Subsystem Overview**:
  - [🇷🇺 Русский](localization/README.ru.md) | [🇬🇧 English](localization/README.en.md)
  - Subsystem architecture, lock-free state storage (`AtomicU8`), bootloader cmdline flags, font rasterizer, and keyboard layout management.
- **Localization Guide**:
  - [🇷🇺 Русский](localization/translation-guide.ru.md) | [🇬🇧 English](localization/translation-guide.en.md)
  - Step-by-step guide for integrating a new language: message specification for translators and technical manual for kernel developers (exhaustive pattern matching, 8×16 bitmap font derivation, PS/2 scancode mapping).

### 3. Development Roadmap ([`roadmap/`](roadmap/))
- **Development Roadmap and Milestone Status**:
  - [🇷🇺 Русский](roadmap/roadmap.ru.md) | [🇬🇧 English](roadmap/roadmap.en.md)
  - 7 developmental phases from hardware bring-up to Ring 3 userland and the WSH command shell.

### 4. Developer Guides ([`guides/`](guides/))
- **Building, Booting, and Debugging**:
  - [🇷🇺 Русский](guides/development.ru.md) | [🇬🇧 English](guides/development.en.md)
  - Toolchain prerequisites, kernel compilation, hybrid ISO creation, QEMU virtualization flags, and COM1 serial diagnostics.
