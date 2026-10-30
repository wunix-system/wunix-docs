[ 🇷🇺 Русский ](development.ru.md) | [ 🇬🇧 English ](development.en.md)

# Сборка, запуск и отладка Wunix

Настоящий документ содержит исчерпывающее техническое руководство по настройке рабочего окружения, компиляции ядра и пользовательских утилит, подготовке файловых образов, формированию гибридного загрузочного ISO-образа и запуску операционной системы Wunix в эмуляторе QEMU.

---

## 1. Системные требования и инструментарий

Сборка ядра Wunix Core и пользовательских программ осуществляется на 64-битных Linux хост-системах (x86_64).

### Необходимые пакеты хоста:
- **Rust Toolchain**: ветка `nightly` с компонентом `rust-src` для сборки `no_std` компонентов и ядра с флагом `build-std`.
- **Target `x86_64-unknown-none`**: целевая голая архитектура без стандартной библиотеки ОС.
- **QEMU (`qemu-system-x86_64`)**: эмулятор целевой аппаратной платформы.
- **xorriso**: утилита создания гибридных образов дисков ISO 9660 (El Torito / UEFI).
- **dosfstools (`mkfs.vfat`)**: утилита форматирования разделов FAT32 для виртуального диска.
- **make, git, python3, build-essential**: вспомогательные утилиты сборки и разметки образов.

### Установка зависимостей (Debian / Ubuntu):
```bash
sudo apt-get update
sudo apt-get install -y qemu-system-x86 xorriso make git build-essential dosfstools python3
rustup toolchain install nightly
rustup default nightly
rustup target add x86_64-unknown-none
rustup component add rust-src --toolchain nightly
```

---

## 2. Быстрый запуск системы (`tools/run.sh`)

В репозитории Wunix подготовлен автоматизированный сборочный конвейер — скрипт [`wunix/tools/run.sh`](file:///home/valdis/my-os/wunix/tools/run.sh), выполняющий полный цикл сборки и запуска «в одну команду»:

```bash
cd wunix

# Запуск Wunix на русском языке (по умолчанию):
./tools/run.sh ru

# Запуск Wunix на английском языке:
./tools/run.sh en

# Только сборка ISO-образа и постоянного диска без запуска QEMU:
./tools/run.sh --build-only

# Запуск с аппаратным ускорением KVM и 1 ГБ оперативной памяти:
ENABLE_KVM=1 QEMU_RAM=1G ./tools/run.sh ru
```

### Переменные окружения для тюнинга запуска:
| Переменная | Значение по умолчанию | Описание |
|---|:---:|---|
| `WUNIX_LANG` | `ru` | Язык сообщений ядра и утилит (`ru` или `en`). |
| `QEMU_RAM` | `512M` | Объем выделяемой оперативной памяти виртуальной машины. |
| `QEMU_SMP` | `1` | Число процессорных ядер CPU. |
| `ENABLE_KVM` | `0` | Включение аппаратного ускорения Linux KVM (`1` для ускорения). |

---

## 3. Детальные этапы сборочного конвейера

Если требуется выполнить сборку отдельных компонентов вручную, процесс делится на следующие стадии:

### 3.1. Компиляция ядра Wunix Core
Ядро находится в `wunix/kernel/` и собирается под `x86_64-unknown-none`:
```bash
cd wunix/kernel
cargo build --release
```
Результирующий бинарник помещается по пути:
`wunix/target/x86_64-unknown-none/release/wunix-kernel` (или в `wunix/kernel/target/...`).

### 3.2. Компиляция программ пользовательского пространства (Userland)
Все консольные утилиты размещены в каталоге [`programs/`](file:///home/valdis/my-os/programs/) и компилируются со статическим relocation model:
```bash
cd programs/<имя_утилиты>
RUSTFLAGS="-C relocation-model=static" cargo build --release --target x86_64-unknown-none
```
Собранные исполняемые файлы копируются в директорию корневой файловой системы `initramfs_root/bin/`.

### 3.3. Создание виртуального жесткого диска (`build/disk.img`)
Для верификации блочного драйвера `virtio-blk` и файловой системы FAT32 создается сырой образ постоянного диска размером 64 МиБ с разметкой MBR:
```bash
# 1. Создание пустого образа:
qemu-img create -f raw build/disk.img 64M

# 2. Запись сигнатуры MBR (раздел FAT32 LBA с сектора 2048 размером 63 МиБ):
python3 -c "
import struct
with open('build/disk.img', 'r+b') as f:
    f.seek(446)
    f.write(struct.pack('<BBBBBBBBII', 0x80, 0, 1, 0, 0x0c, 0, 0, 0, 2048, 129024))
    f.seek(510)
    f.write(b'\x55\xaa')
"

# 3. Форматирование раздела 1 в файловую систему FAT32:
mkfs.vfat -F 32 --offset=2048 build/disk.img
```

### 3.4. Упаковка корневого архива Initramfs (POSIX ustar)
Файлы директории `initramfs_root` упаковываются в блочный архив формата TAR без сжатия:
```bash
cd build/initramfs_root
tar --format=ustar -cf ../iso_root/boot/initramfs.tar *
```

### 3.5. Сборка гибридного ISO-образа через xorriso и Limine
Сформированный каталог `iso_root` компонуется в гибридный загрузочный образ:
```bash
xorriso -as mkisofs -b boot/limine-bios-cd.bin \
    -no-emul-boot -boot-load-size 4 -boot-info-table \
    --efi-boot boot/limine-uefi-cd.bin \
    -efi-boot-part --efi-boot-image --protective-msdos-label \
    build/iso_root -o build/wunix.iso

# Развертывание загрузочного сектора Limine для BIOS/MBR:
tools/limine-bin/limine bios-install build/wunix.iso
```

---

## 4. Конфигурация запуска в эмуляторе QEMU

Полная командная строка запуска QEMU со всем аппаратно-виртуальным стеком (VirtIO, сеть, накопители, дисплей):

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

### Описание аппаратных флагов:
- `-M q35`: Чипсет Intel Q35 с нативной поддержкой шин PCI Express, ACPI и контроллеров APIC.
- `-cdrom build/wunix.iso`: Подключение загрузочного носителя Wunix.
- `-serial stdio`: Перенаправление диагностического порта COM1 UART в терминал хоста.
- `-device virtio-gpu-pci`: Виртуальный графический акселератор VirtIO GPU.
- `-netdev user,id=net0 -device virtio-net-pci,netdev=net0`: Сетевой адаптер `virtio-net` с подключением к виртуальному маршрутизатору QEMU SLIRP (`10.0.2.15`).
- `-drive ... -device virtio-blk-pci`: Блочный диск `virtio-blk` (`/dev/vda`), предоставляющий персистентное хранилище.

---

## 5. Методы отладки ядра

### 5.1. Системные логи COM1 UART
Все вызовы макроса ядра `serial_println!` и функции `dmesg` выводятся в поток STDOUT хост-терминала. Это гарантирует сохранение сообщений об ошибках даже в случае критического отказа графической подсистемы.

### 5.2. Отладка через GDB
Для пошаговой отладки ядра добавьте в параметры запуска QEMU флаги `-s -S`:
- `-s`: открывает GDB-сервер на TCP-порту `1234`.
- `-S`: останавливает процессор на первой инструкции BIOS до загрузки ядра.

```bash
# В отдельном терминале:
gdb wunix/target/x86_64-unknown-none/debug/wunix-kernel
(gdb) target remote :1234
(gdb) break _start
(gdb) continue
```
