[ 🇷🇺 Русский ](README.ru.md) | [ 🇬🇧 English ](README.en.md)

# Документация операционной системы Wunix

Настоящий каталог содержит официальную техническую документацию независимой 64-битной UNIX-подобной операционной системы **Wunix**, разрабатываемой с нуля на языке программирования Rust.

---

## Структура документации

Документация сгруппирована по тематическим разделам. Каждый технический документ поддерживается на русском и английском языках в едином строгом инженерном стиле:

### 1. Архитектура и системный дизайн ([`architecture/`](architecture/))
- **[Общая архитектура системы Wunix](architecture/overview.ru.md)** ([English](architecture/overview.en.md)):
  - Философия проекта, слои абстракции (Applications, Desktop, Base, System API, Core, Hardware), система пакетов (WPM/WSP) и циклы релизов.
- **[Архитектура ядра Wunix Core](architecture/kernel.ru.md)** ([English](architecture/kernel.en.md)):
  - 64-битное Long Mode ядро, модель Higher-Half, протокол Limine, изоляция Ring 3, планировщик задач Round-Robin (квант 50 мс), модульная SRP-структура кодовой базы.
- **[Виртуальная файловая система (VFS) и гибридная иерархия путей](architecture/vfs.ru.md)** ([English](architecture/vfs.en.md)):
  - Единое дерево VFS, прозрачная проекция дисков `C:\`, дескрипторы `FileHandle`, in-memory RamFS, блочный уровень `BlockDevice` / MBR, драйвер FAT32, псевдо-ФС ProcFS и виртуальный реестр Windows NT.
- **[Системные вызовы ядра и Dual ABI](architecture/syscalls.ru.md)** ([English](architecture/syscalls.en.md)):
  - Высокопроизводительный шлюз MSR LSTAR `syscall`/`sysret`, спецификация всех 60+ системных вызовов Linux/POSIX и 24 вызовов Windows NT ABI (SSN).
- **[Системные библиотеки и рантайм](architecture/libraries.ru.md)** ([English](architecture/libraries.en.md)):
  - Пользовательская библиотека `libwunix`, страница ядра vDSO / Syslib (`0x7FFF_FFFF_0000`), зависимости ядра (Kernel Crates) и дорожная карта SDK (`libwui`, `libc`, Rust `std`).
- **[Подсистема драйверов и оборудования](architecture/drivers.ru.md)** ([English](architecture/drivers.en.md)):
  - Диспетчер `DeviceManager`, автоматическое сканирование шины PCI, стек виртуализации VirtIO (`blk`, `net`, `gpu`, `input`), UART 16550 COM1, клавиатура PS/2 с переключением RU/EN и CMOS RTC.
- **[Сетевой стек ядра Wunix](architecture/network.ru.md)** ([English](architecture/network.en.md)):
  - Интеграция `smoltcp` с драйвером `virtio-net`, BSD-сокеты VFS, протоколы Ethernet, ARP, IPv4, ICMP, UDP, TCP и сетевые утилиты (`ping`, `curl`, `ip`, `ss`).

### 2. Справочник man-страниц утилит ([`man/`](man/))
- **[Руководство по man-страницам и каталог команд](man/README.ru.md)** ([English](man/README.en.md)):
  - Использование команды `man`, справочник 28 утилит пользователя Раздела 1 (`cat`, `ls`, `curl`, `wsh`, `top`...) и 10 утилит системного администрирования Раздела 8 (`fdisk`, `mount`, `ip`, `ping`, `init`...).

### 3. Подсистема интернационализации ([`localization/`](localization/))
- **[Обзор подсистемы локализации ядра](localization/README.ru.md)** ([English](localization/README.en.md)):
  - Архитектура интернационализации, атомарное переключение языка, параметры `cmdline` (`lang=ru`/`lang=en`), растровый знакогенератор 8×16 и раскладки клавиатуры.
- **[Руководство разработчика по локализации](localization/translation-guide.ru.md)** ([English](localization/translation-guide.en.md)):
  - Правила добавления новых языков, исчерпывающее сопоставление ключей `MsgKey`, макросы `tr!` и `t!`.

### 4. Дорожная карта разработки ([`roadmap/`](roadmap/))
- **[Дорожная карта и статус этапов Wunix](roadmap/roadmap.ru.md)** ([English](roadmap/roadmap.en.md)):
  - Сводная таблица этапов, статус завершенных фаз 1–10 (включая многопользовательскую безопасность, учетные записи и права VFS) и дорожная карта фаз 11–12 (импорт драйверов Redox OS и аудит железа).

### 5. Руководство разработчика ([`guides/`](guides/))
- **[Сборка, запуск и отладка Wunix](guides/development.ru.md)** ([English](guides/development.en.md)):
  - Системные требования, автоматический запуск через `tools/run.sh`, компиляция программ userland, создание постоянного диска FAT32, конфигурация эмулятора QEMU со стеком VirtIO и отладка через GDB/COM1.
