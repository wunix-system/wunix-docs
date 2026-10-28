[ 🇷🇺 Русский ](README.ru.md) | [ 🇬🇧 English ](README.en.md)

# Wunix Localization Subsystem

This document specifies the architecture and technical mechanisms behind the localization subsystem of the **Wunix** operating system kernel.

---

## 1. Architectural Foundations

The Wunix kernel operates in a freestanding (`no_std`) bare-metal environment and delivers full localization during early hardware bring-up before storage devices and filesystems are mounted.

Key architectural characteristics:

1. **Lock-Free Atomic Language Storage**:
   The active language identifier is stored in a global `AtomicU8` variable ([`kernel/src/i18n/mod.rs`](file:///home/valdis/my-os/wunix/kernel/src/i18n/mod.rs)). This guarantees safe, non-blocking read access from any execution context, including hardware interrupt handlers (ISRs), eliminating deadlock hazards.

2. **Typed Kernel Message Catalog (`MsgKey` and `tr`)**:
   All core system messages are strongly typed via the `MsgKey` enumeration. The Rust compiler enforces compile-time exhaustiveness (`exhaustive match`): if a translation is missing for any key under a registered language, compilation terminates with an error.

3. **Inline Translation Helper (`t`)**:
   The `t(ru, en)` inline function provides fast, zero-allocation substitution for short paired labels in formatted kernel outputs.

4. **Dual Keyboard Layout Support**:
   The PS/2 keyboard driver ([`kernel/src/drivers/keyboard.rs`](file:///home/valdis/my-os/wunix/kernel/src/drivers/keyboard.rs)) supports runtime layout toggling via `Alt+Shift` while ensuring uniform control sequence handling (`Ctrl+C`) irrespective of the active language.

5. **8×16 Monospaced Bitmap Font Rasterizer**:
   Text rendering ([`kernel/src/drivers/font.rs`](file:///home/valdis/my-os/wunix/kernel/src/drivers/font.rs)) uses 8×16 pixel raster glyph matrices. The rasterizer supports standard ASCII, full Cyrillic (including `Ё`/`ё` and `№`), and arbitrary national character sets.

---

## 2. Kernel Command-Line Parameters (cmdline)

The kernel resolves the target interface language at early boot via Limine's `ExecutableFileRequest`.

Supported flags:
- `lang=<code` (e.g., `lang=ru`, `lang=en`, `lang=de`);
- `locale=<code>` (e.g., `locale=ru_RU`, `locale=en_US`, `locale=de_DE`);
- Short positional tokens (`ru`, `en`, `de`).

Menu entries in `limine.conf`:

```ini
/Wunix OS (Russian)
    protocol: limine
    kernel_path: boot():/boot/wunix-kernel
    cmdline: lang=ru

/Wunix OS (English)
    protocol: limine
    kernel_path: boot():/boot/wunix-kernel
    cmdline: lang=en
```

---

## 3. Translation Guides

- **[Wunix Localization Guide (RU)](translation-guide.ru.md)** — Detailed specification and step-by-step instructions in Russian.
- **[Wunix Localization Guide (EN)](translation-guide.en.md)** — Detailed specification and step-by-step instructions in English.

---

## 4. Subsystem Source Files

- [`kernel/src/i18n/mod.rs`](file:///home/valdis/my-os/wunix/kernel/src/i18n/mod.rs) — `Language` enum, `MsgKey` catalog, `tr()` and `t()` dispatchers, `init_from_cmdline()` parser.
- [`kernel/src/drivers/font.rs`](file:///home/valdis/my-os/wunix/kernel/src/drivers/font.rs) — 8×16 font rasterizer, ASCII and Cyrillic bitmap tables, `get_glyph()` function.
- [`kernel/src/drivers/keyboard.rs`](file:///home/valdis/my-os/wunix/kernel/src/drivers/keyboard.rs) — PS/2 keyboard driver, scancode translation, `Alt+Shift` layout switcher.
