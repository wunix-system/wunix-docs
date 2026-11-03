# Создание пользовательских программ для Wunix

Все пользовательские утилиты операционной системы Wunix разрабатываются на языке Rust в режиме `#![no_std]` с использованием системной библиотеки рантайма [`libs/libwunix`](file:///home/valdis/my-os/libs/libwunix).

---

## 1. Анатомия пользовательского пакета

Каждая программа располагается в отдельном каталоге [`programs/<имя_утилиты>/`](file:///home/valdis/my-os/programs/) и представляет собой независимый Git-репозиторий и Cargo-крейт:

```
programs/myutil/
├── Cargo.toml      # Конфигурация крейта и зависимости
├── README.md       # Описание утилиты
├── .gitignore      # Игнорирование артефактов target/
└── src/
    └── main.rs     # Точка входа _start и логика программы
```

### Пример `Cargo.toml`:
```toml
[package]
name = "myutil"
version = "0.1.0"
edition = "2021"

[dependencies]
libwunix = { path = "../../libs/libwunix" }

[profile.release]
opt-level = "z"     # Оптимизация по размеру бинарника
lto = true          # Link-Time Optimization
panic = "abort"     # Завершение без раскрутки стека
```

---

## 2. Минимальный шаблон `src/main.rs`

Поскольку стандартная библиотека Rust (`std`) отсутствует, программа объявляет `#![no_std]` и `#![no_main]`, а точка входа `_start` связывается с загрузчиком ELF:

```rust
#![no_std]
#![no_main]

use libwunix::{println, eprintln, exit};

#[no_mangle]
pub extern "C" fn _start(argc: isize, argv: *const *const u8) -> ! {
    if argc > 1 {
        // Чтение первого аргумента
        let arg1_ptr = unsafe { *argv.offset(1) };
        let arg1_str = unsafe { core::ffi::CStr::from_ptr(arg1_ptr as *const i8) };
        if let Ok(text) = arg1_str.to_str() {
            println!("Аргумент передан: {}", text);
        }
    } else {
        println!("Привет из пространства пользователя Wunix (Ring 3)!");
    }

    exit(0);
}

#[panic_handler]
fn panic(_info: &core::panic::PanicInfo) -> ! {
    eprintln!("Аварийная остановка программы myutil!");
    exit(1);
}
```

---

## 3. Системные вызовы через `libwunix`

Библиотека `libwunix` инкапсулирует ассемблерную инструкцию `syscall` архитектуры x86_64 и предоставляет удобный типобезопасный интерфейс:

### Файловый ввод-вывод:
```rust
use libwunix::{open, close, read, write, O_RDONLY, O_WRONLY, O_CREAT};

let fd = open("/etc/hostname\0".as_ptr(), O_RDONLY, 0);
if fd >= 0 {
    let mut buf = [0u8; 64];
    let n = read(fd as u64, buf.as_mut_ptr(), buf.len() as u64);
    if n > 0 {
        let _ = write(1, buf.as_ptr(), n as u64);
    }
    close(fd as u64);
}
```

### Мультиязычный вывод:
```rust
use libwunix::i18n::t;
use libwunix::println;

// Текст автоматически адаптируется под файл /etc/locale
println!("{}", t("Операция выполнена успешно", "Operation completed successfully"));
```

### Сетевые сокеты:
```rust
use libwunix::{socket, connect, AF_INET, SOCK_STREAM, SockAddrIn};

let sock = socket(AF_INET, SOCK_STREAM, 0);
// Работа с TCP соединением
```

---

## 4. Сборка и установка утилиты

Чтобы скомпилировать и установить новую утилиту в системный образ Wunix:
1. Выполните компиляцию бинарного файла для платформы ядра:
   ```bash
   cargo build --release --target x86_64-unknown-none
   ```
2. Разместите полученный ELF64-файл из `target/x86_64-unknown-none/release/myutil` в каталог `bin/` при формировании архива `initramfs.tar`.
3. Создайте справочное руководство `man1/myutil.1` для вызова `man myutil`.
