# Аппаратные драйверы ядра Wunix OS (Hardware Driver Matrix)

Данный документ содержит полный реестр аппаратных драйверов ядра Wunix OS, сгруппированных по категориям подсистем, с указанием поддерживаемого оборудования, чипсетов, стандартов, шинных интерфейсов и статуса реализации в ядре.

---

## 1. Сетевые интерфейсы (Wired & Wireless Network)

| Драйвер | Категория | Поддерживаемые чипсеты / PCI ID | Протокол / Спецификация | Статус | Расположение в коде |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **intel-ixgbe** | 10GbE Ethernet | Intel 82599ES (`8086:10fb`), X540-T1/T2 (`8086:1528`), X550-T1/T2 (`8086:1563`), 82598EB (`8086:10b6`) | 10GBASE-T, SFP+, 16-байтные Advanced DMA дескрипторы, MSI-X | **Активен** | `kernel/src/drivers/net/ixgbe/` |
| **broadcom-tg3**| 1GbE Ethernet | Broadcom BCM5700..5704, BCM5720 (`14e4:165f`), BCM5721 (`14e4:1659`), BCM5722 (`14e4:165a`), BCM5751/5752, BCM57765, BCM57780 | 1000BASE-T, Tigon3 DMA Ring, NVRAM MAC | **Активен** | `kernel/src/drivers/net/tg3/` |
| **rtl8125** | 2.5GbE Ethernet | Realtek RTL8125, RTL8125B, RTL8125D (`10ec:8125`, `10ec:3000`) | 2.5GBASE-T / NBase-T, 16-байтные дескрипторы RX/TX, DMA 64-бит | **Активен** | `kernel/src/drivers/net/rtl8125/` |
| **rtl8168** | 1GbE Ethernet | Realtek RTL8168, RTL8111, RTL8169, RTL8136 (`10ec:8168`, `10ec:8169`, `10ec:8111`) | 1000BASE-T PCI Express, Dual Ring DMA, C+ Mode | **Активен** | `kernel/src/drivers/net/rtl8168/` |
| **rtl8139** | Fast Ethernet | Realtek RTL8139A/B/C/D, RTL8139+, D-Link DFE-538TX (`10ec:8139`) | 10/100BASE-TX, 64 КБ циклический DMA-буфер | **Активен** | `kernel/src/drivers/net/rtl8139/` |
| **intel-e1000** | 1GbE / 2.5GbE | Intel 82540EM (`8086:100e`), 82545EM, 82574L, I210, I211-AT, I225-V/LM, I226-V/LM | 1000BASE-T/2.5G, Intel PRO/1000 & igb кольца дескрипторов | **Активен** | `kernel/src/drivers/e1000/` |
| **intel-iwlwifi**| Wi-Fi 6/6E/7 | Intel Wireless AX200 (`8086:2723`), AX201 (`8086:06f0`), AX210 (`8086:2725`), Killer AX1650 (`8086:272b`), 9260, 8265, 7265 | IEEE 802.11ax/ac/n, Flow Handler DMA, Station Scan, Frame Injection | **Активен** | `kernel/src/drivers/net/iwlwifi/` |
| **wifi-stack** | 802.11 Core | Универсальный парсер кадров IEEE 802.11 (Management, Beacon, Probe, Data) | 802.11 MAC Frame, SSID Tags, WPA2-PSK / WPA3-SAE | **Активен** | `kernel/src/drivers/net/wifi/` |
| **usb-cdc-net** | USB Ethernet | USB CDC-ECM, CDC-NCM, Android Tethering, Realtek RTL8152/RTL8153, ASIX AX88179 | USB Class 0x02/0x0A, Bulk In/Out Transfers | **Активен** | `kernel/src/drivers/usb/classes/cdc_net.rs` |
| **virtio-net** | VirtIO Network | KVM / QEMU / Proxmox VirtIO Network Adapter | VirtIO Modern/Legacy Network, Multiqueue, Checksum Offload | **Активен** | `kernel/src/drivers/virtio/net.rs` |
| **vmxnet3** | VMware Network | VMware vSphere / ESXi / Workstation VMXNET3 Adapter | Multi-Ring DMA, Interrupt Moderation | **Активен** | `kernel/src/drivers/virt/vmware/vmxnet3.rs` |
| **hyperv-netvsc**| Hyper-V Network | Microsoft Hyper-V / Azure Synthetic Network Adapter | VMBus NetVSC / RNDIS | **Активен** | `kernel/src/drivers/virt/hyperv/netvsc.rs` |
| **xen-netfront** | Xen Network | Citrix Xen / XCP-ng Paravirtualized Network Frontend | Grant Tables, Event Channels | **Активен** | `kernel/src/drivers/virt/xen/netfront.rs` |

---

## 2. USB Стек и контроллеры (Universal Serial Bus)

| Драйвер | Категория | Поддерживаемое оборудование | Протокол / Спецификация | Статус | Расположение в коде |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **xhci** | USB 3.x Host | Контроллеры Intel xHCI (PCH 100..700), AMD Promontory/Ryzen, ASMedia ASM1142/ASM2142/ASM3142, QEMU/Red Hat xHCI | xHCI 1.0..1.2, TRB Command/Transfer/Event Rings, Input Contexts, 64-бит DMA | **Активен** | `kernel/src/drivers/usb/xhci/` |
| **ehci** | USB 2.0 Host | Чипсеты Intel ICH7..ICH10, PCH 5..9 series, AMD SB700..SB900, VIA VT6212 | EHCI 1.0, Async Queue Heads, Periodic Frame List, 480 Мбит/с High-Speed | **Активен** | `kernel/src/drivers/usb/ehci/` |
| **ohci** | USB 1.1 Host | Чипсеты AMD / ATI, NVIDIA nForce, SiS, ALi, Apple Mac I/O | OHCI 1.0a, Endpoint Descriptors (ED) / Transfer Descriptors (TD), 12 Мбит/с | **Активен** | `kernel/src/drivers/usb/ohci/` |
| **uhci** | USB 1.1 Host | Чипсеты Intel PIIX3, PIIX4, ICH0..ICH10, VIA VT82C586 | UHCI 1.1, Frame List Pointers, Queue Heads (QH) / Transfer Descriptors (TD) | **Активен** | `kernel/src/drivers/usb/uhci/` |
| **usb-storage** | USB Накопители | Любые USB-флешки, внешние жесткие диски (USB HDD/SSD), кардридеры USB | USB Mass Storage Class (0x08), SCSI Bulk-Only Transport (BOT), Read10/Write10 | **Активен** | `kernel/src/drivers/usb/classes/storage.rs` |
| **usb-hid** | USB Ввод | Любые проводные и радио USB-клавиатуры и мыши (Logitech, Razer, Microsoft, A4Tech) | USB HID Class (0x03), Subclass Boot (0x01), Keycode Decoder, Relative Mouse | **Активен** | `kernel/src/drivers/usb/classes/hid.rs` |
| **usb-audio** | USB Звук | USB-гарнитуры (HyperX, SteelSeries), USB-микрофоны (Blue Yeti, Rode), внешние ЦАПы | USB Audio Class (UAC 1.0 / UAC 2.0, Class 0x01), Isochronous Transfers, PCM 48kHz | **Активен** | `kernel/src/drivers/usb/classes/audio.rs` |
| **usb-hub** | USB Концентраторы | Встроенные и внешние USB Hubs (USB 2.0 / USB 3.0 хабы) | USB Class 0x09, Hub Descriptors, Port Power & Reset Management | **Активен** | `kernel/src/drivers/usb/classes/hub.rs` |
| **usb-serial** | USB COM-порты | FTDI FT232R/H, WCH CH340, Silicon Labs CP2102, Prolific PL2303 | USB-to-UART адаптеры, настройка скорости 9600..115200 бод, FIFO буфер | **Активен** | `kernel/src/drivers/usb/classes/serial.rs` |

---

## 3. Ввод и периферия (Input, Touchpad & Sensors)

| Драйвер | Категория | Поддерживаемое оборудование | Протокол / Спецификация | Статус | Расположение в коде |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **i2c-designware**| I2C Контроллер | Intel PCH LPSS I2C (Core 6th..14th gen), AMD Ryzen FCH I2C Controller | Synopsys DesignWare I2C Master, Fast Mode 400 кГц, 7-битная адресация | **Активен** | `kernel/src/drivers/i2c/designware.rs` |
| **i2c-hid** | Тачпады ноутбуков | Тачпады и сенсорные экраны Synaptics, Elan, Goodix, Focaltech в ноутбуках Lenovo, ASUS, HP, Dell | Microsoft HID over I2C Protocol v1.0, Absolute Multi-Touch, Tip Switch | **Активен** | `kernel/src/drivers/input/i2c_hid.rs` |
| **ps2-intellimouse**| Мыши PS/2 | Стандартные 3-кнопочные мыши PS/2, мыши с колесиком прокрутки Microsoft IntelliMouse, тачпады Synaptics | 8042 AUX Port, IRQ 12, волшебная последовательность 200-100-80 Hz, 4-байтные пакеты | **Активен** | `kernel/src/drivers/input/ps2_mouse.rs` |
| **ps2-keyboard** | Клавиатуры PS/2 | Стандартные клавиатуры PC/AT PS/2, контроллер Intel 8042 | Скан-коды Set 1/2, IRQ 1, раскладки RU/EN, переключение Alt+Shift | **Активен** | `kernel/src/drivers/keyboard.rs` |
| **virtio-input** | VirtIO Ввод | QEMU / KVM виртуальная клавиатура, планшет и мышь | Linux evdev протокол (EV_KEY, EV_REL, EV_ABS) поверх Virtqueue | **Активен** | `kernel/src/drivers/virtio/input.rs` |
| **vmmouse** | VMware Мышь | Абсолютный указатель мыши VMware Workstation / ESXi | Backdoor I/O координатная синхронизация | **Активен** | `kernel/src/drivers/virt/vmware/vmmouse.rs` |
| **vboxmouse** | VirtualBox Мышь | Абсолютная интеграция указателя мыши VirtualBox | VMMDev HGSMI передача координат курсора | **Активен** | `kernel/src/drivers/virt/vbox/mouse.rs` |

---

## 4. Накопители данных (Storage & Block Devices)

| Драйвер | Категория | Поддерживаемое оборудование | Протокол / Спецификация | Статус | Расположение в коде |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **nvme** | NVMe SSD | Высокоскоростные SSD накопители M.2 / PCIe (Samsung 970/980/990, WD Black, Kingston, Micron) | NVM Express 1.x..2.x, Admin Queue, I/O Submission/Completion Queues, PRP1/PRP2 DMA | **Активен** | `kernel/src/drivers/nvme/` |
| **ahci** | SATA Накопители | SATA жесткие диски и SSD (Intel ICH/PCH, AMD SB/FCH, Marvell, ASMedia) | Serial ATA AHCI 1.0..1.3.1, HBA Command List, Command Table, PRDT DMA | **Активен** | `kernel/src/drivers/ahci/` |
| **ata-piix** | IDE / PATA | Старые накопители PATA/IDE, контроллеры Intel PIIX3/PIIX4, QEMU/VirtualBox IDE | ATA/ATAPI-6, PIO Data Transfer, LBA28 / LBA48 адресация | **Активен** | `kernel/src/drivers/ata.rs` |
| **sdhci** | SD/eMMC Ридеры | Встроенные кардридеры ноутбуков (PCIe SD Class 08:05, Realtek, Genesys, O2Micro, eMMC) | SD Host Controller Spec v1.0..4.20, CMD0..CMD24, 512-байтные блоки | **Активен** | `kernel/src/drivers/sdhci/` |
| **usb-storage** | Внешние диски | Любые USB-флешки и переносные SSD/HDD накопители | SCSI BOT / Transparent Command Set, блочное устройство `/dev/sda` | **Активен** | `kernel/src/drivers/usb/classes/storage.rs` |
| **virtio-blk** | VirtIO Диск | Высокопроизводительные виртуальные диски KVM / QEMU / Proxmox (`/dev/vda`) | VirtIO Block Protocol, Indirect Descriptors, Single/Batch I/O | **Активен** | `kernel/src/drivers/virtio/blk.rs` |
| **virtio-scsi** | VirtIO SCSI | Виртуальные контроллеры SCSI QEMU (`/dev/sda`) | SCSI Architecture Model поверх Virtqueue | **Активен** | `kernel/src/drivers/virtio/scsi.rs` |
| **pvscsi** | VMware SCSI | Paravirtual SCSI контроллер VMware ESXi | Ring Request/Completion, SG DMA | **Активен** | `kernel/src/drivers/virt/vmware/pvscsi.rs` |
| **hyperv-storvsc**| Hyper-V Диск | Виртуальные диски Microsoft Hyper-V / Azure | VMBus StorVSC / SCSI over VMBus | **Активен** | `kernel/src/drivers/virt/hyperv/storvsc.rs` |
| **xen-blkfront** | Xen Диск | Paravirtualized Block Frontend Citrix Xen | Shared Memory Grant Ring, Event Channels | **Активен** | `kernel/src/drivers/virt/xen/blkfront.rs` |

---

## 5. Аудиоподсистема (Audio & Sound)

| Драйвер | Категория | Поддерживаемое оборудование | Протокол / Спецификация | Статус | Расположение в коде |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **snd_hda** | Intel HD Audio | Кодеки Realtek ALC887/ALC892/ALC1220/ALC4080, Intel ICH6..PCH, AMD Azalia, QEMU HDA | High Definition Audio 1.0a, CORB/RIRB кольца глаголов, переключение питания D0 | **Активен** | `kernel/src/drivers/audio/hda/` |
| **snd_ac97** | AC'97 Audio | Чипсеты Intel 82801AA..ICH7, VIA, SiS, VirtualBox AC97 | Audio Codec '97 2.3, Buffer Descriptor List (BDL), Bus Master DMA | **Активен** | `kernel/src/drivers/audio/ac97/` |
| **snd_sb16** | ISA Звук | Creative Sound Blaster 16, ISA PNP, QEMU SB16 | DSP команды 0xB0/0xC0, 16-битный ISA DMA канал 5 контроллера Intel 8237A | **Активен** | `kernel/src/drivers/audio/sb16/` |
| **usb_audio** | USB Звук | USB-гарнитуры, внешние ЦАПы, USB микрофоны | USB Audio Class (UAC 1.0/2.0), Isochronous Endpoint Audio Streaming | **Активен** | `kernel/src/drivers/usb/classes/audio.rs` |

---

## 6. Системная платформа, прерывания и питание (Platform, Power & ACPI)

| Подсистема | Категория | Оборудование / Технология | Описание реализации | Статус | Расположение в коде |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **acpi_core** | Платформа | Таблицы RSDP, XSDT, RSDT, FADT, MADT | Программный парсер таблиц ACPI, валидация контрольных сумм | **Активен** | `kernel/src/drivers/acpi/` |
| **acpi_power** | Управление питанием | Порты PM1a_CNT / PM1b_CNT, FADT Reset Reg | Аппаратное выключение питания (Soft Off S5), сброс и перезагрузка CPU | **Активен** | `kernel/src/drivers/acpi/power.rs` |
| **power_supply** | Батарея ноутбука | ACPI Battery / AC Adapter sysfs интерфейс | Мониторинг сети и емкости батареи в `/sys/class/power_supply/` | **Активен** | `kernel/src/drivers/acpi/power.rs` |
| **lapic** | Прерывания | Local APIC в процессорах Intel Core / Xeon / AMD Ryzen | Spurious Vector, EOI, делитель частоты таймера APIC, TPR | **Активен** | `kernel/src/arch/x86_64/apic/lapic.rs` |
| **ioapic** | Прерывания | I/O APIC чипсета материнской платы (24+ линии) | Маршрутизация прерываний (Redirection Tables), Polarity, Trigger Mode | **Активен** | `kernel/src/arch/x86_64/apic/ioapic.rs` |
| **pci_msi** | Прерывания | Message Signaled Interrupts (MSI & MSI-X) | Настройка адресного регистра 0xFEE00000 и вектора прерывания PCI устройств | **Активен** | `kernel/src/drivers/pci/msi.rs` |
| **pci_bus** | Системная шина | Шина PCI / PCI Express Mechanism #1 (порты 0xCF8/0xCFC) | Сканирование топологии 256 шин, детекция BAR-памяти и I/O | **Активен** | `kernel/src/drivers/pci/` |
| **rtc_cmos** | Системное время | Motorola MC146818 CMOS RTC (порты 0x70/0x71) | Чтение даты, времени, BCD декодирование, 24-часовой формат | **Активен** | `kernel/src/drivers/rtc.rs` |
| **pit8254** | Системный таймер | Intel 8254 PIT (порты 0x40/0x43) | Генерация тиков частотой 100 Гц (квант 10 мс) для планировщика Round-Robin | **Активен** | `kernel/src/arch/x86_64/timer.rs` |
| **i8259_pic** | Прерывания | Dual 8259 PIC (порты 0x20/0x21, 0xA0/0xA1) | Каскадное ремапирование векторов 32..47 | **Активен** | `kernel/src/arch/x86_64/interrupts.rs` |
| **uart16550** | Последовательный порт| COM-порт UART 16550 (0x3F8, COM1) | Скорость 115200 бод, 8N1, FIFO буферизация ввода/вывода | **Активен** | `kernel/src/drivers/serial.rs` |

---

## 7. Безопасность и аппаратная криптография (Security & Hardware RNG)

| Драйвер | Категория | Оборудование | Описание реализации | Статус | Расположение в коде |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **tpm2_crb** | Модуль доверия | TPM 2.0 CRB (Command Response Buffer, Intel PTT / AMD fTPM) | Запрос Locality 0, отправка команд `TPM2_Startup`, `TPM2_GetRandom` | **Активен** | `kernel/src/drivers/tpm/` |
| **cpu_rdrand** | Аппаратный ГПСЧ | Инструкции процессоров Intel Ivy Bridge+ / AMD Zen+ | Генератор случайных чисел `rdrand` / `rdseed` на физических квантовых шумах | **Активен** | `kernel/src/drivers/rng/rdrand.rs` |

---

## 8. Видеоадаптеры и графический вывод (Display & Graphics)

| Драйвер | Категория | Поддерживаемое оборудование | Разрешение / Режим | Статус | Расположение в коде |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **limine_fb** | Графика UEFI/BIOS | Любой UEFI GOP / VESA BIOS фреймбуфер, предоставленный загрузчиком Limine | 32 bpp, любое нативное разрешение монитора (до 4K UHD), ANSI VT100 терминал | **Активен** | `kernel/src/drivers/display.rs` |
| **bochs_vga** | QEMU / Bochs VGA | Стандартный графический адаптер Bochs VBE / QEMU VGA (`1234:1111`) | VBE Dispi интерфейс (порты 0x1CE/0x1CF), смена разрешений | **Активен** | `kernel/src/drivers/virt/qemu/bochs_vga.rs` |
| **virtio_gpu** | Виртуальный GPU | 2D/3D ускоритель QEMU / KVM / Proxmox | Virtqueue 2D Resource Create / Flush / Set Scanout | **Активен** | `kernel/src/drivers/virtio/gpu.rs` |
| **vboxvideo** | VirtualBox Видео | VirtualBox Graphics Adapter VBoxSVGA / VBoxVGA | HGSMI буфер команд, бесшовная интеграция экрана | **Активен** | `kernel/src/drivers/virt/vbox/video.rs` |
| **vmw_svga** | VMware SVGA | VMware SVGA II Display Adapter | FIFO Command Queue, смена видеорежимов | **Активен** | `kernel/src/drivers/virt/vmware/svga.rs` |
| **hyperv_fb** | Hyper-V Видео | Microsoft Hyper-V Synthetic Video | VMBus синтетический линейный видеоадаптер | **Активен** | `kernel/src/drivers/virt/hyperv/synthvid.rs` |
