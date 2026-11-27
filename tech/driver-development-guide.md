# Руководство разработчика драйверов Wunix OS

Wunix использует **гибридную модель драйверов**: критичный к задержкам и привилегированному железу слой (PCI, APIC/IOAPIC/MSI, DMA, AHCI/NVMe, сетевой MAC/PHY, USB-хосты xHCI/EHCI/OHCI/UHCI, HDA) работает в пространстве ядра, а файловые системы, USB-классы, аудио-микширование и оконный композитинг легко выносятся в user space через `libwunix::drv` и общие контракты `wunix-drv`.

---

## 1. Общий контракт шины `wunix-drv` (Ring 0 / Ring 3 + Mock-тестирование)

Крейт `wunix-drv` (`wunix/libs/wunix-drv/`) не зависит от ядра или `std` (`#![no_std]`) и предоставляет:
* **`BusIo`**: единый трейт 8/16/32/64-битного чтения и записи регистров + `wait_mask32(...)`.
* **`MockBusIo<N>`**: программный эмулятор регистров для модульных тестов на хосте (Linux) без QEMU.
* **`DriverDescriptor` / `PciMatchId` / `UsbMatchId`**: метаданные драйвера (имя, версия, автор, лицензия, таблица VID:DID / Class).
* **`abi::{fuse, usbfs, audio, display, uio}`**: бинарные структуры и `ioctl`-константы, общие для ядра и пользовательского пространства.

Один и тот же код драйвера поверх `<B: BusIo>`:
1. Проходит unit-тесты на хосте с `MockBusIo`;
2. Работает в ядре поверх `MmioRegion` / `PioRegion` (`kernel/src/drivers/io.rs`);
3. Работает в пользовательском процессе поверх `UioDevice` (`/dev/uioN`).

---

## 2. Написание ядерного драйвера (`kernel/src/drivers/sdk.rs`)

Ядро предоставляет декларативную регистрацию PCI-драйверов через `register_pci_driver(PciDriverEntry)`:

```rust
use crate::drivers::sdk::{ProbeContext, PciDriverEntry, register_pci_driver};
use wunix_drv::{DriverCategory, DriverDescriptor, DriverError, PciMatchId};

static DESCRIPTOR: DriverDescriptor = DriverDescriptor {
    name: "my_pci_dev",
    version: "0.1.0",
    author: "Vendor Team",
    license: "MIT",
    category: DriverCategory::Other,
};

static IDS: &[PciMatchId] = &[PciMatchId::vendor_device(0x1234, 0x5678)];

fn probe(ctx: &mut ProbeContext<'_, '_>) -> Result<(), DriverError> {
    let mut bar0 = ctx.map_mmio_bar(0)?;
    let _id = bar0.read_u32(0x00);
    // Для многостраничного DMA используйте RAII `DmaPages::alloc(pages)`
    Ok(())
}

pub fn init() {
    register_pci_driver(PciDriverEntry {
        descriptor: &DESCRIPTOR,
        match_ids: IDS,
        probe,
    });
}
```

Все динамически зарегистрированные драйверы автоматически опрашиваются при PCI enumeration и попадают в `/proc/modules` и `/proc/devices`.

---

## 3. Написание user-space драйверов (`libwunix::drv`)

### 3.1. Файловая система FUSE (`/dev/fuse`)

Реализуйте трейт `libwunix::drv::fuse::FuseOps` и запустите цикл обработки поверх `FuseChannel`:

```rust
use libwunix::drv::fuse::{FuseChannel, FuseOps};

struct MyFs;
impl FuseOps for MyFs { /* lookup, getattr, open, read, write, readdir... */ }

let ch = FuseChannel::open_dev()?;
ch.mount_on(b"/mnt/myfs\0")?;
let mut rx = [0u8; 8192];
let mut tx = [0u8; 8192];
let mut fs = MyFs;
loop {
    ch.serve_step(&mut fs, &mut rx, &mut tx)?;
}
```

Внутри host unit-тестов можно вызывать `dispatch_fuse_packet(&mut fs, &req, &mut resp)` напрямую без ядра.

### 3.2. USB-драйвер поверх `usbfs` (`/dev/bus/usb/BBB/DDD`)

Поверх трейта `UsbfsTransport` доступны готовые классы `UsbMassStorageUserDriver` и `UsbHidUserDriver`, а также сырые `control_transfer`, `bulk_transfer` и `interrupt_transfer`:

```rust
use libwunix::drv::usbfs::{UsbfsDevice, UsbfsTransport, UsbHidUserDriver};

let usb = UsbfsDevice::open(b"/dev/bus/usb/001/002\0")?;
let info = usb.device_info()?;
let mut hid = UsbHidUserDriver::new(usb, info.ep_intr_in);
hid.set_boot_protocol(0)?;
let (buttons, dx, dy) = hid.poll_mouse_report(50)?;
```

### 3.3. Аудио-сервер (`/dev/snd/pcm0`, `/dev/dsp`)

Клиент `AudioDeviceClient` управляет частотой/каналами/громкостью через `SNDCTL_DSP_*` и отправляет PCM-фреймы, а `AudioMixer<N>::mix_s16le` выполняет программное микширование с клиппингом и масштабированием громкости.

### 3.4. Оконный композитор (`/dev/fb0`, `/dev/shm`, `/dev/input/eventN`)

1. Откройте `/dev/fb0` через `FramebufferClient::open_fb0()` (выполняет `FBIOGET_VSCREENINFO` + zero-copy `mmap(MAP_SHARED)`).
2. Открывайте буферы клиентских окон через `ShmSurface::open_or_create(b"/dev/shm/win1\0", bytes)`.
3. Читайте события клавиатуры/мыши через `InputEventReader::open(b"/dev/input/event0\0")`.
4. Объединяйте видимые окна функцией `compose_damage_rect(...)` и отправляйте `fb.flush_rect(dirty)`.

### 3.5. Безопасный MMIO + IRQ драйвер поверх UIO (`/dev/uioN`)

```rust
use libwunix::drv::uio::UioDevice;
use wunix_drv::BusIo;

let mut dev = UioDevice::open(b"/dev/uio0\0")?;
let status = dev.read_u32(0x04);
dev.enable_irq()?;
let irq_seq = dev.wait_irq()?;
```

---

## 4. Запуск тестов драйверов

* Host unit-тесты SDK и эмуляции (`wunix-drv` + `libwunix::drv`):
  ```bash
  cd /tmp && cargo test --manifest-path /home/valdis/my-os/wunix/libs/wunix-drv/Cargo.toml --target x86_64-unknown-linux-gnu
  cd /tmp && cargo test --manifest-path /home/valdis/my-os/wunix/libs/libwunix/Cargo.toml --target x86_64-unknown-linux-gnu
  ```
* Ядерные unit-тесты в QEMU (`#[test_case]`):
  ```bash
  python3 wunix/tools/test_kernel_units.py
  ```
* Интеграционный тест гибридной архитектуры и аппаратуры QEMU (`xHCI + usb-kbd + usb-mouse + usb-storage + intel-hda + nvme + virtio-gpu + sdhci + rtl8139`):
  ```bash
  python3 wunix/tools/test_hybrid_drivers.py
  ```
