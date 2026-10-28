[ 🇷🇺 Русский ](development.ru.md) | [ 🇬🇧 English ](development.en.md)

# Сборка, запуск и отладка Wunix

Настоящий документ содержит инструкции по настройке сборочного окружения, компиляции ядра Wunix, формированию загрузочного ISO-образа и запуску в эмуляторе QEMU.

---

## 1. Системные требования и зависимости

Сборка ядра осуществляется на хост-системах Linux (x86_64). Необходимы следующие компоненты:

- **Rust Nightly**: компилятор языка Rust с поддержкой `build-std` и кросс-компиляции без стандартной библиотеки.
- **Компонент `rust-src`**: исходный код стандартных библиотек (`core`, `compiler_builtins`, `alloc`).
- **QEMU (`qemu-system-x86_64`)**: эмулятор целевой платформы.
- **xorriso**: утилита формирования гибридных образов ISO 9660 (El Torito / UEFI).
- **make & git**: для сборки бинарных компонентов загрузчика Limine.

### Установка зависимостей в дистрибутивах Debian / Ubuntu:

```bash
sudo apt-get update
sudo apt-get install -y qemu-system-x86 xorriso make git build-essential
rustup toolchain install nightly
rustup default nightly
rustup component add rust-src --toolchain nightly
```

---

## 2. Компиляция ядра

Исходный код ядра находится в директории `wunix/kernel/`. Проект сконфигурирован для сборки под целевую платформу `x86_64-unknown-none`.

```bash
cd wunix/kernel

# Проверка синтаксиса и типов:
cargo check

# Компиляция ядра в отладочной конфигурации:
cargo build

# Компиляция с оптимизациями:
cargo build --release
```

Результат сборки помещается по пути:
`wunix/kernel/target/x86_64-unknown-none/debug/wunix-kernel` (или `release/wunix-kernel`).

---

## 3. Формирование загрузочного ISO-образа

Wunix использует загрузчик **Limine v8.x** с поддержкой протокола загрузки Higher-Half ядра.

### Структура директории `iso_root`:

```
iso_root/
├── boot/
│   ├── wunix-kernel         # Скомпилированный бинарный образ ядра
│   ├── limine.conf          # Конфигурация меню и параметры cmdline
│   ├── limine-bios.sys      # Системный модуль Limine BIOS
│   ├── limine-bios-cd.bin   # Загрузочный сектор El Torito для CD/DVD
│   └── limine-uefi-cd.bin   # Загрузочный сектор UEFI CD
└── EFI/
    └── BOOT/
        ├── BOOTX64.EFI      # Загрузчик x86_64 UEFI
        └── BOOTIA32.EFI     # Загрузчик IA-32 UEFI
```

### Генерация гибридного ISO через `xorriso`:

```bash
xorriso -as mkisofs -b boot/limine-bios-cd.bin \
    -no-emul-boot -boot-load-size 4 -boot-info-table \
    --efi-boot boot/limine-uefi-cd.bin \
    -efi-boot-part --efi-boot-image --protective-msdos-label \
    iso_root -o build/wunix.iso
```

### Развертывание BIOS MBR на образе:

```bash
limine bios-install build/wunix.iso
```

---

## 4. Запуск в эмуляторе QEMU

Запуск сформированного образа выполняется вызовом эмулятора `qemu-system-x86_64`:

```bash
qemu-system-x86_64 \
    -M q35 \
    -m 512M \
    -smp 1 \
    -cpu qemu64 \
    -cdrom build/wunix.iso \
    -serial stdio
```

### Ключевые параметры QEMU:

| Параметр | Назначение |
|---|---|
| `-M q35` | Использование современного чипсета Intel Q35 (рекомендуется для PCIe и APIC). |
| `-m 512M` | Объем выделяемой оперативной памяти (минимум 256M для ядра с кучей). |
| `-smp <N>` | Количество ядер виртуального процессора. |
| `-cpu <модель>` | Модель CPU (`qemu64`, `max`, `host`). |
| `-enable-kvm` | Аппаратная виртуализация KVM (при наличии прав на `/dev/kvm`). |
| `-serial stdio` | Перенаправление порта COM1 UART в стандартный ввод/вывод терминала. |
| `-display none` | Отключение графического окна (режим работы исключительно через serial). |

---

## 5. Отладка и вывод последовательного порта (COM1)

Драйвер UART 16550 ([`kernel/src/drivers/serial.rs`](file:///home/valdis/my-os/wunix/kernel/src/drivers/serial.rs)) инициализирует порт COM1 по адресу `0x3F8` со скоростью 115200 бод, 8N1.

Все сообщения макроса `serial_println!` направляются в COM1, что позволяет просматривать низкоуровневые диагностические сообщения ядра в консоли хоста параллельно с графическим фреймбуфером.
