[ 🇷🇺 Русский ](development.ru.md) | [ 🇬🇧 English ](development.en.md)

# Building, Booting, and Debugging Wunix

This document specifies the setup of the host build environment, Wunix kernel compilation, bootable ISO image packaging, and QEMU emulator execution.

---

## 1. System Requirements and Dependencies

Kernel compilation is supported on x86_64 Linux hosts. The following software tools are required:

- **Rust Nightly**: Rust compiler toolchain supporting `build-std` and freestanding bare-metal compilation.
- **`rust-src` component**: Source code for core standard crates (`core`, `compiler_builtins`, `alloc`).
- **QEMU (`qemu-system-x86_64`)**: Target x86_64 platform emulator.
- **xorriso**: ISO 9660 filesystem manipulation utility supporting El Torito and UEFI boot catalogs.
- **make & git**: Required to compile the host Limine bootloader utility.

### Dependency Installation on Debian / Ubuntu:

```bash
sudo apt-get update
sudo apt-get install -y qemu-system-x86 xorriso make git build-essential
rustup toolchain install nightly
rustup default nightly
rustup component add rust-src --toolchain nightly
```

---

## 2. Kernel Compilation

The kernel source tree resides in `wunix/kernel/`. The project compiles for the `x86_64-unknown-none` target.

```bash
cd wunix/kernel

# Syntax and type validation:
cargo check

# Compile debug kernel binary:
cargo build

# Compile optimized release binary:
cargo build --release
```

The resulting ELF Higher-Half kernel binary is produced at:
`wunix/kernel/target/x86_64-unknown-none/debug/wunix-kernel` (or `release/wunix-kernel`).

---

## 3. Bootable ISO Packaging

Wunix integrates the **Limine v8.x** bootloader supporting the Limine Higher-Half boot protocol.

### `iso_root` Directory Structure:

```
iso_root/
├── boot/
│   ├── wunix-kernel         # Compiled kernel binary
│   ├── limine.conf          # Bootloader configuration and cmdline flags
│   ├── limine-bios.sys      # Limine BIOS system module
│   ├── limine-bios-cd.bin   # El Torito CD/DVD boot sector
│   └── limine-uefi-cd.bin   # UEFI CD boot sector
└── EFI/
    └── BOOT/
        ├── BOOTX64.EFI      # x86_64 UEFI bootloader executable
        └── BOOTIA32.EFI     # IA-32 UEFI bootloader executable
```

### Hybrid ISO Generation with `xorriso`:

```bash
xorriso -as mkisofs -b boot/limine-bios-cd.bin \
    -no-emul-boot -boot-load-size 4 -boot-info-table \
    --efi-boot boot/limine-uefi-cd.bin \
    -efi-boot-part --efi-boot-image --protective-msdos-label \
    iso_root -o build/wunix.iso
```

### Installing BIOS MBR onto ISO:

```bash
limine bios-install build/wunix.iso
```

---

## 4. Execution in QEMU Emulator

To run the generated hybrid ISO in QEMU:

```bash
qemu-system-x86_64 \
    -M q35 \
    -m 512M \
    -smp 1 \
    -cpu qemu64 \
    -cdrom build/wunix.iso \
    -serial stdio
```

### Core QEMU Flags:

| Flag | Function |
|---|---|
| `-M q35` | Modern Intel Q35 chipset emulation (standard for PCIe and APIC). |
| `-m 512M` | Allocated guest physical memory (256M minimum for heap operation). |
| `-smp <N>` | Number of virtual processor cores. |
| `-cpu <model>` | Emulated CPU architecture (`qemu64`, `max`, `host`). |
| `-enable-kvm` | Hardware-assisted KVM acceleration (requires `/dev/kvm` host permissions). |
| `-serial stdio` | Redirects COM1 UART serial output directly to host terminal. |
| `-display none` | Disables graphics window (headless serial-only mode). |

---

## 5. Serial Port Diagnostics (COM1)

The UART 16550 driver ([`kernel/src/drivers/serial.rs`](file:///home/valdis/my-os/wunix/kernel/src/drivers/serial.rs)) initializes port COM1 at I/O base `0x3F8` at 115200 baud (8N1).

All `serial_println!` kernel macros write directly to COM1, enabling capture and inspection of early kernel diagnostics on the host terminal simultaneously with framebuffer display.
