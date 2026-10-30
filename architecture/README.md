# Архитектура Wunix / Architecture

[ 🇷🇺 Русский ](overview.ru.md) | [ 🇬🇧 English ](overview.en.md)

Раздел содержит подробную техническую документацию подсистем ядра и системной архитектуры операционной системы Wunix:

- **1. Общая архитектура системы / System Overview**:
  - [🇷🇺 Русский (RU)](overview.ru.md) | [🇬🇧 English (EN)](overview.en.md)
  - Слои операционной системы, модель пакетов WPM/WSP, жизненный цикл релизов.
- **2. Архитектура ядра Wunix Core / Kernel Architecture**:
  - [🇷🇺 Русский (RU)](kernel.ru.md) | [🇬🇧 English (EN)](kernel.en.md)
  - Higher-Half модель, изоляция Ring 3, планировщик задач, vDSO/Syslib, структура каталогов ядра.
- **3. Виртуальная файловая система / Virtual File System (VFS)**:
  - [🇷🇺 Русский (RU)](vfs.ru.md) | [🇬🇧 English (EN)](vfs.en.md)
  - Гибридное дерево, трансляция путей `C:\`, Inode, дескрипторы `FileHandle`, FAT32, ProcFS, реестр Windows NT.
- **4. Системные вызовы и Dual ABI / System Calls**:
  - [🇷🇺 Русский (RU)](syscalls.ru.md) | [🇬🇧 English (EN)](syscalls.en.md)
  - Аппаратный шлюз MSR LSTAR, спецификация всех 60+ Linux/POSIX сисколлов и 24 вызовов Windows NT ABI.
- **5. Системные библиотеки и рантайм / Libraries & Runtime**:
  - [🇷🇺 Русский (RU)](libraries.ru.md) | [🇬🇧 English (EN)](libraries.en.md)
  - Библиотека `libwunix`, страница vDSO / Syslib, крейты ядра, дорожная карта SDK (`libwui`, `libc`, `std`).
- **6. Подсистема драйверов и устройств / Hardware & Drivers**:
  - [🇷🇺 Русский (RU)](drivers.ru.md) | [🇬🇧 English (EN)](drivers.en.md)
  - `DeviceManager`, шина PCI, стек драйверов VirtIO (`blk`, `net`, `gpu`, `input`), UART 16550, PS/2 клавиатура, RTC.
- **7. Сетевой стек / Network Stack**:
  - [🇷🇺 Русский (RU)](network.ru.md) | [🇬🇧 English (EN)](network.en.md)
  - Архитектура на базе `smoltcp` и `virtio-net`, BSD-сокеты, протоколы TCP/UDP/ICMP, утилиты `ping`, `curl`, `ip`, `ss`.
