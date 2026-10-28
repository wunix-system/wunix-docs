[ 🇷🇺 Русский ](overview.ru.md) | [ 🇬🇧 English ](overview.en.md)

# Wunix System Architecture Overview

**Wunix** is an independent UNIX-like operating system written from scratch in the Rust programming language.

---

## 1. Design Goals and Principles

The project focuses on creating a modern, safe, and extensible general-purpose platform:

1. **Memory Safety by Design**:
   - The use of Rust eliminates at compile-time the typical vulnerabilities inherent to C/C++: use-after-free, buffer overflows, data races, and double-free errors.
   - Low-level `unsafe` blocks are strictly isolated within hardware abstraction layers (`arch/`) and drivers.
2. **Hybrid Modular Kernel Architecture**:
   - Monolithic execution performance combined with modular subsystem isolation.
   - Device drivers and filesystems communicate with the kernel through typed traits.
3. **UNIX Philosophy and POSIX Compliance**:
   - "Everything is a file" model through Virtual File System (VFS) abstractions.
   - Distinct separation of responsibilities among utilities and composable stream pipelines.
4. **Clean Slate Ecosystem Without Legacy Burden**:
   - Free of 16-bit and 32-bit legacy backward-compatibility shims. The kernel operates exclusively in 64-bit Long Mode (`x86_64`).
5. **Native Localization**:
   - First-class localization support embedded into kernel architecture and user interfaces.

---

## 2. Operating System Layers

The Wunix architecture is structured into 6 interconnected layers:

```
┌────────────────────────────────────────────────────────┐
│                   User Applications                    │
│                 (Wunix Applications)                   │
├────────────────────────────────────────────────────────┤
│                Desktop Environment & UI                │
│              (Wunix Desktop & Wunix UI)                │
├────────────────────────────────────────────────────────┤
│                 Base Userland System                   │
│                    (Wunix Base)                        │
├────────────────────────────────────────────────────────┤
│                 System Interface & API                 │
│               (System Calls / POSIX)                   │
├────────────────────────────────────────────────────────┤
│                Operating System Kernel                 │
│                    (Wunix Core)                        │
├────────────────────────────────────────────────────────┤
│                   Hardware Platform                    │
│             (x86_64 CPU, RAM, MMIO, I/O)               │
└────────────────────────────────────────────────────────┘
```

### Layer Roles:

| Layer | Functional Scope |
|---|---|
| **Wunix Applications** | Application software: text editors, web browsers, file managers, developer toolchains, terminal emulators. |
| **Wunix Desktop & UI** | Graphical environment: window compositor, status bar, font renderer, and Wunix UI component library. |
| **Wunix Base** | Core userland environment (analogous to GNU Userland): libc/wlibc system libraries, WSH (Wunix Shell), and core utilities (`ls`, `cat`, `grep`, `cp`). |
| **System API** | Fast system call boundary (`syscall` / `sysret`) featuring pointer validation across Ring 3 / Ring 0. |
| **Wunix Core** | Kernel: process scheduler, virtual memory management (Paging), hardware drivers, VFS, and concurrency primitives. |
| **Hardware** | Physical hardware or virtualized platforms (QEMU/KVM). |

---

## 3. Package Management Architecture

To decouple the base operating system from third-party application software, Wunix provides two dedicated package managers:

### WSP (Wunix System Package)
- Dedicated exclusively to operating system maintenance.
- Manages updates to the kernel (`wunix-kernel`), system libraries, and core daemons.
- Supports atomic transaction state and safe recovery points (System Rollback).

### WPM (Wunix Package Manager)
- Userland application package manager (analogous to `apt`, `pacman`, or `cargo`).
- Handles installation, removal, and dependency resolution of end-user programs from Wunix software repositories.
- Enforces cryptographic package signatures and sandboxed dependencies.

---

## 4. Release Model and Development Branches

The project follows a four-tier release lifecycle:

| Branch | Purpose | Stability Level |
|---|---|---|
| `unstable` | Active subsystem development and experimental drivers. | Experimental, subject to rapid changes. |
| `testing` | Pre-release stabilization and integration testing. | High. |
| `stable` | Production releases for end-users and developers. | Production-grade, strict change validation. |
| `LTS` | Long Term Support branch. | Maximum reliability, critical security fixes only. |
