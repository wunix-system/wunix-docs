# Сборка и запуск Wunix

В этом документе описаны требования к инструментарию хост-системы, структура процесса сборки и запуск операционной системы Wunix в эмуляторе QEMU или на реальном оборудовании.

---

## 1. Требования к хост-системе

Сборка Wunix осуществляется на операционных системах Linux (Ubuntu, Debian, Fedora, Arch Linux).

### Необходимые пакеты:
```bash
# Ubuntu / Debian
sudo apt update
sudo apt install -y build-essential git curl xorriso qemu-system-x86 mtools nasm parted

# Arch Linux
sudo pacman -S base-devel git curl xorriso qemu-system-x86 mtools nasm parted
```

### Настройка тулчейна Rust:
Для сборки `no_std` ядра требуются нестабильные возможности компилятора (`nightly`), а также компонент `rust-src`:
```bash
rustup default nightly
rustup component add rust-src
```

---

## 2. Процесс сборки и запуска (`tools/run.sh`)

Главной точкой входа для сборки всех компонентов Wunix является скрипт [`wunix/tools/run.sh`](file:///home/valdis/my-os/wunix/tools/run.sh).

### Фазы работы сборочного пайплайна:
1. **Сборка ядра (`wunix-kernel`)**:
   Ядро компилируется для кастомного таргета `x86_64-unknown-none` в режиме Higher-Half с моделью релокации `static` / `kernel`. Исполняемый ELF-файл ядра сохраняется в `target/x86_64-unknown-none/release/wunix-kernel`.
2. **Сборка пользовательских программ (`programs/`)**:
   Все утилиты (`cat`, `ls`, `wsh`, `login`, `fdisk` и др.) компилируются в Ring 3 с линковкой против `libwunix`.
3. **Упаковка Initramfs**:
   Скрипт формирует дерево каталогов (`bin/`, `etc/`, `dev/`, `win/`, `usr/share/man/`), упаковывает его в архив `initramfs.tar` и генерирует системные базы (`/etc/passwd`, `/etc/shadow`, реестр `system.reg`).
4. **Создание загрузочного ISO-образа**:
   С помощью `xorriso` формируется гибридный ISO-образ `build/wunix.iso` с загрузчиком **Limine** (поддержка BIOS MBR и UEFI).
5. **Запуск в эмуляторе QEMU**:
   Запуск виртуальной машины x86_64 с аппаратной виртуализацией KVM (при наличии), сетевым адаптером `virtio-net-pci` и тестовыми блочными дисками.

---

## 3. Режимы запуска скрипта

### Стандартный запуск (Live-сессия на русском языке):
```bash
cd wunix
./tools/run.sh
```

### Запуск с выбором английской локали:
```bash
WUNIX_LANG=en ./tools/run.sh
```

### Только компиляция (без старта QEMU):
```bash
./tools/run.sh --build-only
# или кратко
./tools/run.sh -b
```

### Полная пересборка всех компонентов с нуля:
```bash
./tools/run.sh --rebuild
# или кратко
./tools/run.sh -r
```

---

## 4. Параметры виртуальной машины QEMU

Скрипт запускает QEMU со следующей конфигурацией оборудования:
* **Архитектура**: `qemu-system-x86_64`
* **Процессор и ядра**: `-cpu host` (или `-cpu qemu64`), 2 ядра CPU (`-smp 2`)
* **Оперативная память**: 512 МиБ (`-m 512M`)
* **Графика**: фреймбуфер высокого разрешения 1024x768 (`-vga std`)
* **Сеть**: VirtIO-Net (`-netdev user,id=net0,hostfwd=... -device virtio-net-pci,netdev=net0`)
* **Накопители**:
  * Первичный диск: тестовый жесткий диск 100 МиБ (`build/disk.img` как `/dev/hda`)
  * VirtIO диск: блочное устройство (`build/virtio.img` как `/dev/vda`)
* **COM-порт (Serial UART 16550)**: вывод дублируется в консоль хоста для отладки (`-serial stdio`).
