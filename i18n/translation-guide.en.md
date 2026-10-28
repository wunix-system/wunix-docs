# Guide: How to Translate Wunix into Your Language

This guide explains how to add support for a new language to the **Wunix** operating system.

The document is divided into two distinct tracks:
- **[Part 1: For Translators and Writers (No programming required)](#part-1-for-translators-and-writers-no-programming-required)** — covers the key glossary, screen context, length limitations, tone guidelines, and a ready-to-use template.
- **[Part 2: For Engineers and Developers (Rust source code)](#part-2-for-engineers-and-developers-rust-source-code)** — a step-by-step technical guide for integrating the language into the bare-metal kernel: typing, exhaustive pattern matching, bootloader cmdline parsing, 8×16 bitmap font glyph rasterization, and keyboard scancode mapping.

---

## Localization Architecture at a Glance

Localization in Wunix is embedded directly into the kernel ([`kernel/src/i18n/`](file:///home/valdis/my-os/wunix/kernel/src/i18n/mod.rs)). Unlike userland software on Linux or Windows where translation strings are loaded from external `.mo` or `.json` files, the Wunix kernel boots directly on bare metal without an initial filesystem. Therefore, all core strings are compiled into the kernel binary image.

The kernel provides two complementary localization mechanisms:
1. **Typed Key Catalog (`MsgKey` and `tr` function)** — used for all core operating system messages (system readiness, input prompt, device autoconfiguration). The Rust compiler strictly enforces at build time that every registered language provides translations for all keys.
2. **Pairwise Inline Helper (`t(ru, en)`)** — a fast, lightweight helper for short inline string pairs ("enabled/disabled", "ok/fail").

---

## Part 1: For Translators and Writers (No Programming Required)

If you are not a programmer, you don't need to know Rust to contribute a translation. Your goal is to provide accurate, concise, and context-appropriate translations of system messages.

### 1.1. System Message Context & Length Limits

During boot, Wunix runs in a text/framebuffer mode with a fixed 8×16 pixel font. Overly long strings may cause unwanted line wraps or break column alignment in hardware summary tables.

Below is the complete list of system keys, their meanings, and screen context:

| Key Identifier (`MsgKey`) | English Original | Russian Original | Screen Context & Trigger | Guidelines & Constraints |
|---|---|---|---|---|
| `SystemReady` | `wunix: system ready` | `wunix: система готова к работе` | Printed to the console when hardware bring-up and initialization finish. | Concise status notification. Avoid exclamation marks or celebratory tone. |
| `PromptInput` | `wunix: type something on keyboard (layout switch: Alt+Shift)...` | `wunix: введите текст с клавиатуры (переключение раскладки: Alt+Shift)...` | Prompt line displayed at the bottom of the screen before console echo loop. | Informs user that the system is ready for input and indicates the layout hotkey. |
| `InterruptCancel` | `^C (input cancelled)` | `^C (ввод отменен)` | Printed when user presses `Ctrl+C`. | Line cancellation notice. Conventionally starts with `^C`. |
| `Cpu` | `cpu` | `процессор` | Device label in autoconfiguration summary (dmesg). | Single word, prefer lowercase for consistency with UNIX kernel convention. |
| `Memory` | `memory` | `память` | RAM capacity label in device summary. | Single word, lowercase. |
| `Display` | `display` | `экран` | Video adapter / framebuffer label. | Single word, lowercase. |
| `Timer` | `timer` | `таймер` | System timer label (PIT/APIC). | Single word, lowercase. |
| `Keyboard` | `keyboard` | `клавиатура` | PS/2 keyboard label. | Single word, lowercase. |
| `Uptime` | `uptime` | `время работы` | System tick counter / elapsed time indicator. | 1–2 words. |

### 1.2. Wunix Style Guide

1. **Conciseness (The UNIX Way)**: Kernel messages must be clean, precise, and understated. No marketing talk or extraneous decorative punctuation.
2. **Casing Consistency**: Hardware component tags (`cpu`, `memory`, `timer`) are lowercase; full kernel status messages start with the subsystem identifier (`wunix: ...`).
3. **Alphabet & Special Characters**: If your target language uses diacritics, umlauts, or accented characters (e.g., German `ä`, `ö`, `ü`, `ß` or French `é`, `à`, `ç`), please explicitly list them in your submission so engineers can draw bitmap glyphs for them in the kernel font.

### 1.3. Translation Submission Template

To submit a translation to the Wunix project, copy the block below, fill in your translations, and submit it via a GitHub Issue or Pull Request:

```text
Language Name: [e.g., German / Deutsch]
ISO 639-1 Code: [e.g., de]
Specific Alphabet Characters: [e.g., ä, ö, ü, ß, Ä, Ö, Ü]

MsgKey TRANSLATIONS:
- SystemReady: "..."
- PromptInput: "..."
- InterruptCancel: "..."
- Cpu: "..."
- Memory: "..."
- Display: "..."
- Timer: "..."
- Keyboard: "..."
- Uptime: "..."

COMMON LABELS:
- Enabled / Disabled: "..." / "..."
- Error / Success: "..." / "..."
```

---

## Part 2: For Engineers and Developers (Rust Source Code)

If you are implementing language support directly in the kernel source code, follow these steps.

As a reference example, we will implement support for German (**German**, language code `de`).

### Step 1. Register the Language in `Language` Enum

Open [`kernel/src/i18n/mod.rs`](file:///home/valdis/my-os/wunix/kernel/src/i18n/mod.rs). Add a new variant to the `Language` enum:

```rust
#[derive(Clone, Copy, PartialEq, Eq, Debug)]
#[repr(u8)]
pub enum Language {
    Russian = 0,
    English = 1,
    German = 2, // Newly added language
}
```

> [!NOTE]
> The `#[repr(u8)]` attribute is critical: the active language is stored in a global lockless atomic variable `AtomicU8`. This allows reading the active language without locks or risk of deadlocks, even inside interrupt service routines (ISRs).

Also update `current_language()`:

```rust
pub fn current_language() -> Language {
    match CURRENT_LANGUAGE.load(Ordering::Relaxed) {
        0 => Language::Russian,
        2 => Language::German,
        _ => Language::English,
    }
}
```

---

### Step 2. Add Key Mapping in `tr(key)`

In the same file [`kernel/src/i18n/mod.rs`](file:///home/valdis/my-os/wunix/kernel/src/i18n/mod.rs), add a match arm for the new language in `tr()`:

```rust
pub fn tr(key: MsgKey) -> &'static str {
    match current_language() {
        Language::Russian => match key {
            MsgKey::SystemReady => "wunix: система готова к работе",
            MsgKey::PromptInput => "wunix: введите текст с клавиатуры (переключение раскладки: Alt+Shift)...",
            MsgKey::InterruptCancel => "^C (ввод отменен)",
            MsgKey::Cpu => "процессор",
            MsgKey::Memory => "память",
            MsgKey::Display => "экран",
            MsgKey::Timer => "таймер",
            MsgKey::Keyboard => "клавиатура",
            MsgKey::Uptime => "время работы",
        },
        Language::English => match key {
            MsgKey::SystemReady => "wunix: system ready",
            MsgKey::PromptInput => "wunix: type something on keyboard (layout switch: Alt+Shift)...",
            MsgKey::InterruptCancel => "^C (input cancelled)",
            MsgKey::Cpu => "cpu",
            MsgKey::Memory => "memory",
            MsgKey::Display => "display",
            MsgKey::Timer => "timer",
            MsgKey::Keyboard => "keyboard",
            MsgKey::Uptime => "uptime",
        },
        Language::German => match key {
            MsgKey::SystemReady => "wunix: System ist betriebsbereit",
            MsgKey::PromptInput => "wunix: Text eingeben (Layout wechseln: Alt+Shift)...",
            MsgKey::InterruptCancel => "^C (Eingabe abgebrochen)",
            MsgKey::Cpu => "Prozessor",
            MsgKey::Memory => "Speicher",
            MsgKey::Display => "Bildschirm",
            MsgKey::Timer => "Timer",
            MsgKey::Keyboard => "Tastatur",
            MsgKey::Uptime => "Betriebszeit",
        },
    }
}
```

> [!IMPORTANT]
> **Exhaustive Pattern Matching Guarantee:**
> Do NOT use wildcard `_ => ...` patterns in `tr()`. The Rust compiler enforces exhaustive matching: if another developer adds a new key to `enum MsgKey` in the future, the build will fail immediately with `non-exhaustive patterns` until explicit translations are provided for all registered languages.

---

### Step 3. Parse Kernel Bootloader Flags in `init_from_cmdline`

The Wunix kernel receives command-line parameters from the Limine bootloader via `ExecutableFileRequest`. To parse flags such as `lang=de`, `locale=de_DE`, or `de`, update `init_from_cmdline` in [`kernel/src/i18n/mod.rs`](file:///home/valdis/my-os/wunix/kernel/src/i18n/mod.rs):

```rust
pub fn init_from_cmdline(cmdline: &str) {
    for token in cmdline.split_whitespace() {
        if let Some((k, v)) = token.split_once('=') {
            if k.eq_ignore_ascii_case("lang") || k.eq_ignore_ascii_case("locale") {
                match v {
                    "ru" | "RU" | "ru_RU" | "russian" => set_language(Language::Russian),
                    "en" | "EN" | "en_US" | "english" => set_language(Language::English),
                    "de" | "DE" | "de_DE" | "german"  => set_language(Language::German),
                    _ => {}
                }
            }
        } else if token.eq_ignore_ascii_case("ru") || token.eq_ignore_ascii_case("russian") {
            set_language(Language::Russian);
        } else if token.eq_ignore_ascii_case("en") || token.eq_ignore_ascii_case("english") {
            set_language(Language::English);
        } else if token.eq_ignore_ascii_case("de") || token.eq_ignore_ascii_case("german") {
            set_language(Language::German);
        }
    }
}
```

Add a unit test in `mod tests` verifying that `init_from_cmdline("lang=de")` sets the language correctly.

---

### Step 4. Add 8×16 Font Glyphs for Special Characters

In Wunix, graphical text rendering uses the raster font generator in [`kernel/src/drivers/font.rs`](file:///home/valdis/my-os/wunix/kernel/src/drivers/font.rs).

Each glyph is an array of 16 bytes (`[u8; 16]`):
- Each byte encodes one horizontal row of 8 pixels.
- Most-significant bit 7 (`0x80`) is the leftmost pixel.
- Least-significant bit 0 (`0x01`) is the rightmost pixel.
- Bit value `1` = text foreground color; `0` = background color.

#### Designing Glyph `'ä'` (Lowercase a with umlaut):

```text
8x16 Grid        Bits        Hex     Description
[0]  ........  00000000    0x00    (blank line above character)
[1]  ........  00000000    0x00
[2]  ..#..#..  00100100    0x24    Umlaut dots (left & right)
[3]  ..#..#..  00100100    0x24
[4]  ........  00000000    0x00    (spacing between dots and letter body)
[5]  ..####..  00111100    0x3c    Top arc of 'a'
[6]  .#....#.  01000010    0x42
[7]  ......#.  00000010    0x02
[8]  ..#####.  00111110    0x3e    Middle horizontal stroke
[9]  .#....#.  01000010    0x42
[10] .#....#.  01000010    0x42
[11] ..####.#  00111101    0x3d    Bottom right tail of 'a'
[12] ........  00000000    0x00
[13] ........  00000000    0x00
[14] ........  00000000    0x00
[15] ........  00000000    0x00
```

#### Adding the Glyph to Kernel Source:
In [`kernel/src/drivers/font.rs`](file:///home/valdis/my-os/wunix/kernel/src/drivers/font.rs):

1. Define the bitmap constant:
```rust
pub const GLYPH_A_UMLAUT: [u8; 16] = [
    0x00, 0x00, 0x24, 0x24, 0x00, 0x3c, 0x42, 0x02,
    0x3e, 0x42, 0x42, 0x3d, 0x00, 0x00, 0x00, 0x00,
];
```

2. Register the character in `get_glyph(ch: char)`:
```rust
pub fn get_glyph(ch: char) -> Option<&'static [u8; 16]> {
    match ch {
        ' '..='~' => {
            let idx = (ch as u32 - 0x20) as usize;
            Some(&ASCII_GLYPHS[idx])
        }
        'А'..='я' => {
            let idx = (ch as u32 - 0x0410) as usize;
            Some(&CYRILLIC_GLYPHS[idx])
        }
        'Ё' => Some(&GLYPH_YO_UPPER),
        'ё' => Some(&GLYPH_YO_LOWER),
        '№' => Some(&GLYPH_NUMERO),
        'ä' => Some(&GLYPH_A_UMLAUT), // Newly registered glyph
        _ => None,
    }
}
```

If a character is not registered in `get_glyph`, the kernel safely falls back to `REPLACEMENT_GLYPH` (a clean bounding box), preventing visual corruption.

---

### Step 5. (Optional) Keyboard Layout Support

If the language requires a specialized keyboard layout (such as German QWERTZ or French AZERTY):

1. In [`kernel/src/drivers/keyboard.rs`](file:///home/valdis/my-os/wunix/kernel/src/drivers/keyboard.rs), add a variant to `enum Layout`:
   ```rust
   pub enum Layout {
       English,
       Russian,
       German,
   }
   ```
2. Implement scancode mapping (similar to `map_russian_keycode`).
3. Wire the layout into `DualLayout::map_keycode` and update the active layout tag (`[DE] `).

---

### Step 6. Build, Boot & Test in QEMU

1. **Verify compilation:**
   ```bash
   cd wunix/kernel
   cargo check
   ```

2. **Add a bootloader menu entry in `tools/limine.conf`:**
   ```ini
   /Wunix OS (German)
       protocol: limine
       kernel_path: boot():/boot/wunix-kernel
       cmdline: lang=de
   ```

3. **Boot in QEMU with the new language:**
   The `tools/run.sh` script automates ISO packaging and sets kernel boot arguments:
   ```bash
   ./tools/run.sh de
   # or:
   ./tools/run.sh --lang de
   ```

4. **Verify on screen:**
   - Readiness banner: `wunix: System ist betriebsbereit`.
   - Device autoconfiguration: `wunix: autoconfiguration: Prozessor, Speicher, Bildschirm, Timer, Tastatur`.
   - Input echo & `Ctrl+C` interrupt handling: `^C (Eingabe abgebrochen)`.

---

## Pre-Commit Checklist (Definition of Done)

- [ ] New language variant added to `enum Language` in `kernel/src/i18n/mod.rs`.
- [ ] Every key in `enum MsgKey` is translated in `tr()` without using wildcard `_ => ...`.
- [ ] Boot argument parser `init_from_cmdline()` handles `lang=...` and short names.
- [ ] Non-ASCII characters added to `kernel/src/drivers/font.rs` and handled in `get_glyph()`.
- [ ] `cargo check` in `wunix/kernel/` succeeds with exit code 0 without errors or warnings.
- [ ] The OS boots successfully in QEMU via `./tools/run.sh <lang_code>`.
- [ ] Git commit message follows project standards: `feat(i18n): add German (de) language support`.
