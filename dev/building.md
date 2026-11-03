# Сборка и запуск Wunix

В этом документе описаны требования к сборочному инструментарию хост-системы, пошаговый процесс компиляции подсистем операционной системы Wunix с использованием стандартных инструментов (`cargo`, `make`, `xorriso`, `limine`) и запуск в эмуляторе QEMU.

---

## 1. Требования к сборочному окружению

Сборка компонентов Wunix осуществляется на базе Linux (Ubuntu, Debian, Fedora, Arch Linux).

### Системные пакеты:
```bash
# Ubuntu / Debian
sudo apt update
sudo apt install -y build-essential git curl xorriso qemu-system-x86 mtools nasm

# Arch Linux
sudo pacman -S base-devel git curl xorriso qemu-system-x86 mtools nasm
```

### Настройка тулчейна Rust:
Для сборки `no_std` ядра и системных утилит требуется компилятор Rust nightly с установленной платформой `x86_64-unknown-none` и исходным кодом стандартной библиотеки:
```bash
rustup default nightly
rustup target add x86_64-unknown-none
rustup component add rust-src
```

### Загрузчик Limine:
Для подготовки загрузочного носителя используется официальный релиз загрузчика Limine (ветка `v8.x-binary`):
```bash
git clone https://github.com/limine-bootloader/limine.git --branch=v8.x-binary --depth=1 limine
make -C limine
```

---

## 2. Пошаговая сборка компонентов

Операционная система Wunix построена на модульной архитектуре независимых репозиториев. Каждый компонент собирается штатными средствами своего инструментария.

### 2.1. Сборка ядра (`wunix-kernel`)
Ядро компилируется для архитектуры `x86_64-unknown-none` в режиме Higher-Half:
```bash
cd wunix-kernel
cargo build --release --target x86_64-unknown-none
```
Скомпилированный бинарный файл ядра формата ELF64 формируется по пути:
`target/x86_64-unknown-none/release/wunix-kernel`

### 2.2. Сборка системных библиотек
- **Системная библиотека Rust (`libwunix`)**:
  ```bash
  cd libwunix
  cargo build --release --target x86_64-unknown-none
  ```
- **Стандартная библиотека C (`libcwunix`)**:
  ```bash
  cd libcwunix
  make
  ```
  Формирует статическую библиотеку `lib/libcwunix.a` и стартап-объекты `crt1.o`.

### 2.3. Сборка командной оболочки (`wsh`)
```bash
cd wsh
cargo build --release --target x86_64-unknown-none
```
Исполняемый файл оболочки формируется по пути:
`target/x86_64-unknown-none/release/wsh`

### 2.4. Сборка базовых утилит (`wunix-base`)
Каждая системная утилита (`init`, `login`, `cat`, `ls`, `mount`, `fdisk`, `winstall` и др.) компилируется через Cargo:
```bash
cd wunix-base/<имя_утилиты>
cargo build --release --target x86_64-unknown-none
```
Скомпилированный файл помещается в `target/x86_64-unknown-none/release/<имя_утилиты>`.

---

## 3. Подготовка корневой файловой системы (`initramfs.tar`)

Ядро Wunix загружает начальную корневую файловую систему из стандартного архива формата POSIX tar (USTAR):

```bash
# 1. Создание структуры каталогов VFS
mkdir -p initramfs_root/{bin,etc,dev,tmp,root,home,mnt,boot,usr/share/man}

# 2. Размещение бинарных файлов утилит и оболочки
cp path/to/wunix-base/target/x86_64-unknown-none/release/* initramfs_root/bin/
cp path/to/wsh/target/x86_64-unknown-none/release/wsh initramfs_root/bin/

# 3. Базовые системные конфигурации
cat > initramfs_root/etc/hostname << 'EOF'
wunix
EOF

cat > initramfs_root/etc/locale << 'EOF'
LANG=ru
EOF

# 4. Упаковка в архив формата USTAR
tar -cf initramfs.tar -C initramfs_root .
```

---

## 4. Создание загрузочного ISO-образа

Формирование загрузочного гибридного ISO-образа (поддержка BIOS и UEFI) выполняется с помощью `xorriso` и загрузчика Limine:

```bash
# 1. Подготовка дерева файлов ISO
mkdir -p iso_root/boot iso_root/EFI/BOOT

# 2. Копирование ядра, initramfs и конфигурации загрузчика
cp target/x86_64-unknown-none/release/wunix-kernel iso_root/boot/
cp initramfs.tar iso_root/boot/

cat > iso_root/boot/limine.conf << 'EOF'
timeout: 3

/Wunix OS
    protocol: limine
    kernel_path: boot():/boot/wunix-kernel
    module_path: boot():/boot/initramfs.tar
    cmdline: lang=ru mode=live
EOF

# 3. Размещение загрузочных файлов Limine
cp limine/limine-bios.sys limine/limine-bios-cd.bin limine/limine-uefi-cd.bin iso_root/boot/
cp limine/BOOTX64.EFI limine/BOOTIA32.EFI iso_root/EFI/BOOT/

# 4. Создание ISO-образа через xorriso
xorriso -as mkisofs -b boot/limine-bios-cd.bin \
    -no-emul-boot -boot-load-size 4 -boot-info-table \
    --efi-boot boot/limine-uefi-cd.bin \
    -efi-boot-part --efi-boot-image --protective-msdos-label \
    iso_root -o wunix.iso

# 5. Установка загрузочного сектора BIOS для дисков
./limine/limine bios-install wunix.iso
```

---

## 5. Запуск в эмуляторе QEMU

Запуск сформированного образа выполняется стандартной командой `qemu-system-x86_64`:

```bash
qemu-system-x86_64 \
    -boot d \
    -cdrom wunix.iso \
    -m 512M \
    -smp 2 \
    -cpu qemu64 \
    -netdev user,id=net0 \
    -device virtio-net-pci,netdev=net0 \
    -chardev stdio,id=char0,signal=off \
    -serial chardev:char0
```

### Запуск с аппаратным ускорением KVM:
```bash
qemu-system-x86_64 \
    -enable-kvm -cpu host \
    -boot d \
    -cdrom wunix.iso \
    -m 1G \
    -smp 2 \
    -netdev user,id=net0 \
    -device virtio-net-pci,netdev=net0 \
    -chardev stdio,id=char0,signal=off \
    -serial chardev:char0
```

### Запуск в консольном режиме (headless):
Для тестирования в терминале без графического окна используется параметр `-display none`:
```bash
qemu-system-x86_64 -display none -serial stdio -boot d -cdrom wunix.iso -m 512M
```
