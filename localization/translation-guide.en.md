[ 🇷🇺 Русский ](translation-guide.ru.md) | [ 🇬🇧 English ](translation-guide.en.md)

# Wunix Operating System Localization Guide

This document serves as the official specification and technical guide for integrating new languages into the **Wunix** operating system.

The guide is structured by target audience:
- **[Section 1. Localization Architecture](#1-localization-architecture)** — Architectural foundations of the kernel translation subsystem.
- **[Section 2. Message Specification (For Translators and Linguists)](#2-message-specification-for-translators-and-linguists)** — Kernel message catalog, display context, formatting constraints, and submission template. Programming knowledge is not required.
- **[Section 3. Technical Implementation (For Kernel Developers)](#3-technical-implementation-for-kernel-developers)** — Step-by-step instructions for Rust source code integration, exhaustive pattern matching, 8×16 bitmap font rasterization, and bootloader configuration.
- **[Section 4. Acceptance Criteria](#4-acceptance-criteria)** — Verification checklist required prior to merging changes into the repository.

---

## 1. Localization Architecture

Unlike userland operating system environments where message catalogs (such as GNU gettext `.mo` files) are loaded dynamically from a filesystem, the Wunix kernel operates in a freestanding (`no_std`) bare-metal environment. Localization is performed during early hardware initialization before storage devices and filesystems are mounted.

All system string resources are compiled directly into the kernel binary image. Localization is implemented using two complementary mechanisms:

1. **Typed Key Catalog (`MsgKey` and `tr` function)**:
   Used for core operating system messages. Every system message is identified by a variant of the `MsgKey` enumeration. The Rust compiler enforces compile-time exhaustiveness (`exhaustive match`): omitting a translation for any key under a registered language produces a compilation failure (`error[E0004]: non-exhaustive patterns`).
2. **Pairwise Inline Helper (`t(ru, en)`)**:
   Used for concise inline status labels in formatted kernel output.

The active language is stored in a global `AtomicU8` variable ([`kernel/src/i18n/mod.rs`](file:///home/valdis/my-os/wunix/kernel/src/i18n/mod.rs)), providing lock-free, race-free read access from any execution context, including interrupt service routines (ISRs).

---

## 2. Message Specification (For Translators and Linguists)

Contributing a translation to Wunix does not require software engineering experience. The translator translates the strings according to the functional context and screen constraints, then submits the completed specification table.

### 2.1. Kernel Message Catalog

During early boot, console output is rendered to a fixed-size framebuffer or text console with monospaced 8×16 pixel character cells. Line overflow must be avoided to prevent misalignment of hardware autoconfiguration tables.

| Key (`MsgKey`) | English Source | Russian Source | Functional Context & Trigger | Format & Constraints |
|---|---|---|---|---|
| `SystemReady` | `wunix: system ready` | `wunix: система готова к работе` | Emitted when kernel hardware initialization completes successfully. | Single sentence. The `wunix: ` prefix is required. Formal affirmative tone. |
| `PromptInput` | `wunix: type something on keyboard (layout switch: Alt+Shift)...` | `wunix: введите текст с клавиатуры (переключение раскладки: Alt+Shift)...` | Kernel console input prompt indicating the active layout toggle shortcut. | Prompt string instructing the user that interactive input is active. |
| `InterruptCancel` | `^C (input cancelled)` | `^C (ввод отменен)` | Printed when user input is aborted via `Ctrl+C`. | Standard `^C ` prefix, concise explanation in parentheses. |
| `Cpu` | `cpu` | `процессор` | Central Processing Unit label in the device summary table. | Single noun in lowercase. |
| `Memory` | `memory` | `память` | System RAM capacity label in the autoconfiguration summary. | Single noun in lowercase. |
| `Display` | `display` | `экран` | Video display / framebuffer adapter label. | Single noun in lowercase. |
| `Timer` | `timer` | `таймер` | System timer label (PIT/APIC). | Single noun in lowercase. |
| `Keyboard` | `keyboard` | `клавиатура` | PS/2 input device label. | Single noun in lowercase. |
| `Uptime` | `uptime` | `время работы` | System elapsed time counter label. | 1–2 words in lowercase. |

### 2.2. Style Requirements

1. **Technical Precision**: Terminology must adhere to standard UNIX/POSIX conventions.
2. **Character Casing**: Hardware component labels (`cpu`, `memory`, `timer`) are lowercase to maintain column alignment. Subsystem notifications begin with the `wunix: ` prefix.
3. **Special Character Inventory**: If the target language contains diacritics, umlauts, or non-ASCII characters (e.g., `ä`, `ö`, `ü`, `ß`, `é`, `è`, `ç`, `ñ`), they must be explicitly listed in the submission template. Each character requires a corresponding 16-byte bitmap glyph in the font rasterizer.

### 2.3. Translation Submission Template

To submit a translation, fill out the following template:

```text
Language Name: [English and native name, e.g., German / Deutsch]
ISO 639-1 Code: [Two-letter code, e.g., de]
Alphabet Special Characters: [List all characters outside standard ASCII]

MsgKey Message Catalog:
- SystemReady: "..."
- PromptInput: "..."
- InterruptCancel: "..."
- Cpu: "..."
- Memory: "..."
- Display: "..."
- Timer: "..."
- Keyboard: "..."
- Uptime: "..."

Auxiliary Labels:
- Enabled / Disabled: "..." / "..."
- Error / Success: "..." / "..."
```

---

## 3. Technical Implementation (For Kernel Developers)

Integrating a new language into the kernel source tree involves the following sequential steps. The German language (**German**, code `de`) is used as the reference implementation.

### Step 1. Register Language in `Language` Enum

In [`kernel/src/i18n/mod.rs`](file:///home/valdis/my-os/wunix/kernel/src/i18n/mod.rs), add a new variant to `enum Language`:

```rust
#[derive(Clone, Copy, PartialEq, Eq, Debug)]
#[repr(u8)]
pub enum Language {
    Russian = 0,
    English = 1,
    German = 2, // Newly added language
}
```

The `#[repr(u8)]` attribute is mandatory because the discriminant is stored directly in an `AtomicU8`.

Update the accessor function:

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

### Step 2. Implement Key Mapping in `tr(key)`

In [`kernel/src/i18n/mod.rs`](file:///home/valdis/my-os/wunix/kernel/src/i18n/mod.rs), add a match arm for the new language in `tr()`:

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

> [!NOTE]
> **Exhaustive Pattern Matching Enforcement:**
> Rust enforces exhaustive pattern matching. Using wildcard `_ => ...` patterns inside `match key` is prohibited by engineering guidelines. If a new key is added to `MsgKey` in the future, the compiler will abort the build with `error[E0004]` until explicit translations are provided across all registered languages.

---

### Step 3. Kernel Command-Line Parameter Parsing

The Wunix kernel receives boot parameters from the bootloader. To parse configuration arguments such as `lang=de`, `locale=de_DE`, or the positional token `de`, update `init_from_cmdline` in [`kernel/src/i18n/mod.rs`](file:///home/valdis/my-os/wunix/kernel/src/i18n/mod.rs):

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

---

### Step 4. 8×16 Bitmap Font Glyph Integration

Graphical character rendering is executed by the kernel font rasterizer ([`kernel/src/drivers/font.rs`](file:///home/valdis/my-os/wunix/kernel/src/drivers/font.rs)).

#### Raster Format Specification:
- Cell dimensions: Fixed 8 pixels wide by 16 pixels high.
- Representation: Array of 16 unsigned 8-bit integers (`[u8; 16]`).
- Row layout:
  - Most significant bit (bit 7, `0x80`) maps to the leftmost pixel of the cell row.
  - Least significant bit (bit 0, `0x01`) maps to the rightmost pixel of the cell row.
  - Bit value `1` = foreground pixel; `0` = background pixel.

#### Bitmap Derivation Example: Glyph `'ä'` (Lowercase a with umlaut):

```text
Cell Row              Binary          Hexadecimal
Row 0:   ........    00000000        0x00
Row 1:   ........    00000000        0x00
Row 2:   ..#..#..    00100100        0x24  (umlaut dots)
Row 3:   ..#..#..    00100100        0x24
Row 4:   ........    00000000        0x00
Row 5:   ..####..    00111100        0x3c  (top arc)
Row 6:   .#....#.    01000010        0x42
Row 7:   ......#.    00000010        0x02
Row 8:   ..#####.    00111110        0x3e
Row 9:   .#....#.    01000010        0x42
Row 10:  .#....#.    01000010        0x42
Row 11:  ..####.#    00111101        0x3d  (character tail)
Row 12:  ........    00000000        0x00
Row 13:  ........    00000000        0x00
Row 14:  ........    00000000        0x00
Row 15:  ........    00000000        0x00
```

#### Kernel Integration:
In [`kernel/src/drivers/font.rs`](file:///home/valdis/my-os/wunix/kernel/src/drivers/font.rs):

1. Define the bitmap constant:
```rust
pub const GLYPH_A_UMLAUT: [u8; 16] = [
    0x00, 0x00, 0x24, 0x24, 0x00, 0x3c, 0x42, 0x02,
    0x3e, 0x42, 0x42, 0x3d, 0x00, 0x00, 0x00, 0x00,
];
```

2. Register the character mapping in `get_glyph(ch: char)`:
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
        'ä' => Some(&GLYPH_A_UMLAUT), // Registered character
        _ => None,
    }
}
```

Characters not registered in `get_glyph` safely fall back to `REPLACEMENT_GLYPH` (a monospaced bounding box), preventing renderer corruption.

---

### Step 5. Bootloader Menu Configuration

To enable language selection in the bootloader interface, update `limine.conf`:

```ini
/Wunix OS (German)
    protocol: limine
    kernel_path: boot():/boot/wunix-kernel
    cmdline: lang=de
```

---

### Step 6. Compilation and Verification

1. Type and syntax check:
   ```bash
   cd wunix/kernel
   cargo check
   ```

2. Compile the kernel binary:
   ```bash
   cd wunix/kernel
   cargo build
   ```

---

### Step 7. (Optional) Keyboard Layout Configuration

If the target language requires an alternate keyboard layout (such as German QWERTZ or French AZERTY):

1. In [`kernel/src/drivers/keyboard.rs`](file:///home/valdis/my-os/wunix/kernel/src/drivers/keyboard.rs), add a variant to `enum Layout`:
   ```rust
   pub enum Layout {
       English,
       Russian,
       German,
   }
   ```
2. Implement scancode translation table logic.
3. Integrate the layout into `DualLayout::map_keycode` with `Alt+Shift` cycle support.

---

## 4. Acceptance Criteria

Before merging modifications into the repository, verify that:

- [ ] Variant added to `enum Language` in `kernel/src/i18n/mod.rs`.
- [ ] All keys in `MsgKey` are translated in `tr()` without using wildcard patterns.
- [ ] `init_from_cmdline` parses `lang=<code`, `locale=<code>`, and positional tokens.
- [ ] Non-ASCII characters have 16-byte matrices in `kernel/src/drivers/font.rs` registered in `get_glyph()`.
- [ ] Bootloader menu entry added in `limine.conf`.
- [ ] `cargo check` executes cleanly with zero errors or warnings.
- [ ] Commit message conforms to project standards: `feat(i18n): add German (de) language support`.
