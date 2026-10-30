[ 🇷🇺 Русский ](development.ru.md) | [ 🇬🇧 English ](development.en.md)

# Building, Booting, and Debugging Wunix

This document provides a comprehensive developer guide for environment setup, kernel and userland compilation, disk image preparation, hybrid ISO image creation, and launching Wunix inside QEMU.

---

## 1. Prerequisites and Toolchain Setup

Wunix Core and userland programs are compiled on 64-bit Linux host systems (`x86_64`).

### Required Host Packages:
- **Rust Toolchain**: `nightly` channel with the `rust-src` component for `no_std` kernel builds with `build-std`.
- **Target `x86_64-unknown-none`**: Bare-metal target architecture without standard library runtime.
- **QEMU (`qemu-system-x86_64`)**: Target x86_64 hardware emulator.
- **xorriso**: Generates hybrid ISO 9660 filesystem images (El Torito / UEFI).
- **dosfstools (`mkfs.vfat`)**: Formats FAT32 partitions for the virtual disk.
- **make, git, python3, build-essential**: Build automation and disk scripting tools.

### Installing Dependencies (Debian / Ubuntu):
```bash
sudo apt-get update
sudo apt-get install -y qemu-system-x86 xorriso make git build-essential dosfstools python3
rustup toolchain install nightly
rustup default nightly
rustup target add x86_64-unknown-none
rustup component add rust-src --toolchain nightly
```

---

## 2. Quick Start: One-Command Build and Run (`tools/run.sh`)

The repository includes an end-to-end automation script [`wunix/tools/run.sh`](file:///home/valdis/my-os/wunix/tools/run.sh) handling compilation, packaging, and QEMU execution:

```bash
cd wunix

# Boot Wunix in Russian (default):
./tools/run.sh ru

# Boot Wunix in English:
./tools/run.sh en

# Build ISO image and virtual disk without starting QEMU:
./tools/run.sh --build-only

# Run with KVM hardware acceleration and 1 GB RAM:
ENABLE_KVM=1 QEMU_RAM=1G ./tools/run.sh ru
```

### Environment Variables:
| Variable | Default Value | Description |
|---|:---:|---|
| `WUNIX_LANG` | `ru` | System language for kernel and utilities (`ru` or `en`). |
| `QEMU_RAM` | `512M` | Virtual machine RAM allocation. |
| `QEMU_SMP` | `1` | Number of virtual CPU cores. |
| `ENABLE_KVM` | `0` | Enables Linux KVM hardware virtualization (`1` to enable). |

---

## 3. Step-by-Step Manual Build Pipeline

For isolated builds of specific subsystems, follow these individual steps:

### 3.1. Kernel Compilation
The kernel code resides in `wunix/kernel/` and targets `x86_64-unknown-none`:
```bash
cd wunix/kernel
cargo build --release
```
Binary output: `wunix/target/x86_64-unknown-none/release/wunix-kernel`.

### 3.2. Userland Utilities Compilation
Core utilities reside under [`programs/`](file:///home/valdis/my-os/programs/) and compile with a static relocation model:
```bash
cd programs/<program_name>
RUSTFLAGS="-C relocation-model=static" cargo build --release --target x86_64-unknown-none
```
Binaries are placed into the root filesystem directory `initramfs_root/bin/`.

### 3.3. Virtual Hard Disk Preparation (`build/disk.img`)
To verify the `virtio-blk` driver and FAT32 filesystem, a 64 MiB raw disk image with MBR partition table is created:
```bash
# 1. Create blank raw image:
qemu-img create -f raw build/disk.img 64M

# 2. Write MBR partition table (FAT32 LBA starting at sector 2048, 63 MiB size):
python3 -c "
import struct
with open('build/disk.img', 'r+b') as f:
    f.seek(446)
    f.write(struct.pack('<BBBBBBBBII', 0x80, 0, 1, 0, 0x0c, 0, 0, 0, 2048, 129024))
    f.seek(510)
    f.write(b'\x55\xaa')
"

# 3. Format partition 1 as FAT32:
mkfs.vfat -F 32 --offset=2048 build/disk.img
```

### 3.4. Packaging Initramfs Archive (POSIX ustar)
Package root filesystem files into an uncompressed TAR archive:
```bash
cd build/initramfs_root
tar --format=ustar -cf ../iso_root/boot/initramfs.tar *
```

### 3.5. Building the Hybrid ISO with xorriso and Limine
Assemble the hybrid bootable CD/DVD and USB image:
```bash
xorriso -as mkisofs -b boot/limine-bios-cd.bin \
    -no-emul-boot -boot-load-size 4 -boot-info-table \
    --efi-boot boot/limine-uefi-cd.bin \
    -efi-boot-part --efi-boot-image --protective-msdos-label \
    build/iso_root -o build/wunix.iso

# Deploy BIOS MBR boot sectors:
tools/limine-bin/limine bios-install build/wunix.iso
```

---

## 4. QEMU Launch Parameters

Full command line starting QEMU with all virtualization devices attached:

```bash
qemu-system-x86_64 \
    -M q35 \
    -m 512M \
    -smp 1 \
    -cpu qemu64 \
    -cdrom build/wunix.iso \
    -boot d \
    -serial stdio \
    -device virtio-gpu-pci \
    -device virtio-net-pci,netdev=net0 \
    -netdev user,id=net0 \
    -drive file=build/disk.img,if=none,id=disk0,format=raw \
    -device virtio-blk-pci,drive=disk0 \
    -device virtio-tablet-pci
```

### Hardware Flags Explanation:
- `-M q35`: Intel Q35 chipset providing native PCI Express, ACPI, and APIC controllers.
- `-cdrom build/wunix.iso`: Attaches the bootable ISO media.
- `-serial stdio`: Redirects COM1 UART diagnostics directly to the host shell.
- `-device virtio-gpu-pci`: Accelerated 2D/3D VirtIO GPU graphics adapter.
- `-netdev user,id=net0 -device virtio-net-pci,netdev=net0`: VirtIO network adapter connected via QEMU user SLIRP router (`10.0.2.15`).
- `-drive ... -device virtio-blk-pci`: Persistent block storage drive registered as `/dev/vda`.

---

## 5. Kernel Debugging Techniques

### 5.1. COM1 UART Logs
Kernel diagnostic logs emitted via `serial_println!` and `dmesg` are forwarded to host terminal STDOUT. Diagnostic logs remain preserved even in cases of complete display server halts.

### 5.2. Debugging with GDB
To perform step-by-step assembly and source debugging, attach GDB via `-s -S`:
- `-s`: Opens GDB stub server on TCP port `1234`.
- `-S`: Pauses CPU on the initial instruction prior to bootloader execution.

```bash
# In an independent terminal window:
gdb wunix/target/x86_64-unknown-none/debug/wunix-kernel
(gdb) target remote :1234
(gdb) break _start
(gdb) continue
```
