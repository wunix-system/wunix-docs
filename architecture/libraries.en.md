[ 🇷🇺 Русский ](libraries.ru.md) | [ 🇬🇧 English ](libraries.en.md)

# Wunix System Libraries and Runtime Architecture

The Wunix operating system architecture features a structured library hierarchy: from low-level `no_std` kernel dependencies (`Kernel Crates`) and in-kernel vDSO virtual pages (`Syslib`), to the lightweight userland base library (`libwunix`).

---

## 1. Userland System Library (`libwunix`)

The `libwunix` library ([`wunix/libs/libwunix/`](file:///home/valdis/my-os/wunix/libs/libwunix)) serves as the lightweight `no_std` runtime for all userland command-line utilities and system daemons in Ring 3.

```
libs/libwunix/src/
├── lib.rs            # Raw assembly syscall0..syscall6 wrappers, stdio, basic I/O
├── syslib.rs         # Interface to kernel vDSO (fast string routines and trampolines)
├── errno.rs          # Typed POSIX error enumeration (Errno) and check() helper
├── i18n.rs           # Localization support (parses /etc/locale, provides t())
├── env.rs            # Process environment variables parser (getenv, environ iterator)
├── fmt.rs            # Allocation-free numeric string formatting (utoa, itoa, utoa_hex)
├── str_utils.rs      # Raw C-string routines (strlen, strchr, starts_with_ci, strtoul)
├── time.rs           # Converts Unix Epoch milliseconds into structured DateTime
└── registry.rs       # Client for the Windows NT virtual registry (/win/registry/)
```

### 1.1. Modules and Responsibilities
| Module | Location | Purpose |
|:---|:---|:---|
| `lib.rs` | `libs/libwunix/src/lib.rs` | Direct assembly trampolines `syscall0`..`syscall6`, `print!`, `println!`, `eprintln!` macros, standard handles `STDIN_FILENO`, `STDOUT_FILENO`, `STDERR_FILENO`, file operations `open`, `read`, `write`, `close`, `lseek`, `stat`, `getdents64`, `mount`, `umount2`, `system_poweroff`, `system_reboot`. |
| `syslib.rs` | `libs/libwunix/src/syslib.rs` | Grants safe access to the kernel vDSO page for zero-overhead operations. |
| `errno.rs` | `libs/libwunix/src/errno.rs` | Typed `Errno` enum (`EPERM`, `ENOENT`, `EACCES`, `EINVAL`, etc.) and `check(res)` converting return codes to `Result<u64, Errno>`. |
| `i18n.rs` | `libs/libwunix/src/i18n.rs` | Reads `/etc/locale` and provides `t(ru, en)` string translation with zero runtime overhead. |
| `env.rs` | `libs/libwunix/src/env.rs` | Environment variables management (`envp`): allocation-free `getenv(key)` lookup and `environ()` iterator. |
| `fmt.rs` | `libs/libwunix/src/fmt.rs` | Zero-allocation number formatting into caller-provided stack buffers: `utoa`, `itoa`, `utoa_hex`. |
| `str_utils.rs`| `libs/libwunix/src/str_utils.rs` | Raw string utilities: `strlen`, `memcmp`, `strchr`, case-insensitive `starts_with_ci`, integer parser `strtoul`. |
| `time.rs` | `libs/libwunix/src/time.rs` | Time utilities (`Timespec`, `DateTime`): converts Unix Epoch milliseconds to calendar dates taking leap years into account. |
| `registry.rs` | `libs/libwunix/src/registry.rs` | Windows NT virtual registry client (`reg_query_sz`, `reg_query_dword`) querying `/win/registry/`. |

---

## 2. Virtual Shared Kernel Library (vDSO / Syslib)

The kernel maps a read-only **Syslib / vDSO** page at virtual address **`0x0000_7FFF_FFFF_0000`** into every Ring 3 process ([`kernel/src/task/syslib.rs`](file:///home/valdis/my-os/wunix/kernel/src/task/syslib.rs)):

```
Process Virtual Address: 0x0000_7FFF_FFFF_0000
┌────────────────────────────────────────────────────────┐
│               Structure SyslibSharedInfo               │
│ - Magic signature: 'WUNIXLIB' (8 bytes)                │
│ - ABI Version: 1                                       │
│ - System timer frequency: 100 Hz                       │
│ - Boot timestamp (milliseconds)                        │
│ - Exported symbol and function offset table            │
├────────────────────────────────────────────────────────┤
│        Address: 0x0000_7FFF_FFFF_1000 (vDSO Code)      │
│ 1. Inlined strlen (17 bytes x86_64 assembly)           │
│ 2. Inlined memcpy (using rep movsb instruction)        │
│ 3. Inlined memset (using rep stosb instruction)        │
│ 4. Pre-assembled Linux syscall trampolines             │
│ 5. Pre-assembled NT ABI trampolines                    │
└────────────────────────────────────────────────────────┘
```

### Advantages of vDSO:
- **Zero-Syscall Execution**: `strlen`, `memcpy`, and `memset` execute directly in Ring 3 without triggering privilege level transitions.
- **Independence**: Userland programs execute foundational memory and string routines without relying on an external dynamic linker (`ld.so`).

---

## 3. Kernel Crates and Dependencies

Low-level kernel subsystems in Wunix Core rely on verified, lightweight `no_std` crates:

| Crate | Version | Subsystem Location | Role |
|:---|:---:|:---|:---|
| `limine` | `0.6.5` | `arch/x86_64/` | Bootloader protocol: HHDM, memory map, framebuffer, initramfs modules. |
| `x86_64` | `0.15` | `arch/`, `memory/` | Architectural types: control registers, 4-level paging, GDT, IDT, TSS. |
| `spin` | `0.9` | `sync/`, `fs/`, `task/` | Concurrency primitives: `Mutex`, `RwLock`, and `Once` in `no_std`. |
| `smoltcp` | `0.11` | `net/` | Standalone TCP/IP network stack: Ethernet II, ARP, IPv4, ICMP, UDP, TCP. |
| `uart_16550` | `0.3` | `drivers/serial.rs` | COM1 UART serial port driver (`0x3F8`) for diagnostics. |
| `pic8259` | `0.11` | `arch/x86_64/` | Dual 8259 PIC interrupt controller (remaps IRQs to `32..47`). |
| `linked_list_allocator` | `0.10.5` | `memory/heap.rs` | Dynamic heap allocator enabling `extern crate alloc` (`Box`, `Vec`, `String`). |
| `pc-keyboard` | `0.7` | `drivers/keyboard.rs` | PS/2 keyboard scan code decoding supporting Shift, Alt, Ctrl, and CapsLock. |
| `kernel::i18n` | *Internal* | `i18n/mod.rs` | Kernel localization catalog: `MsgKey` keys, `tr!` and `t!` macros. |

---

## 4. Planned Libraries Roadmap (TODO)

Userland SDKs and libraries scheduled for future development:

1. **`libwui` (Wunix UI SDK)**: Graphical toolkit for windowed applications featuring declarative layouts, event dispatching, and hardware acceleration via `virtio-gpu`.
2. **`libwunix-c` (`libc.so`)**: Standard C library (`stdio.h`, `stdlib.h`, `string.h`, `unistd.h`, `malloc`) for porting POSIX software.
3. **Rust Standard Library (`std`)**: Native Rust `std` support (target `x86_64-unknown-wunix`) covering `std::fs`, `std::net`, `std::thread`, `std::sync`.
4. **`libwsh` (Shell SDK)**: Shell engine library: lexical tokenizer, pipeline graphs, command history, and Tab auto-completion.
5. **`libwcrypto`**: Cryptographic algorithms (SHA-256, SHA-512, Ed25519, ChaCha20) for package signature verification and password security.
6. **`libwpm` / `libwsp`**: Package management engine: `package.toml` parsing, dependency resolution, atomic updates, and system rollback points.
7. **`ntdll.dll` / `libntemu`**: Win32 user-space compatibility layer (`kernel32.dll`, `advapi32.dll`) translating Win32 calls into Wunix NT ABI syscalls.
