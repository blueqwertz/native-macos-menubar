# native-macos-menubar

A Claude Code skill for building native macOS menu bar apps — the kind Apple ships for WiFi, Bluetooth, Battery, and Passwords.

## What it does

Guides Claude agents to produce menu bar apps using only native macOS primitives:

- `MenuBarExtra` / `NSStatusItem` + `NSPopover`
- System materials (`regularMaterial`, `ultraThinMaterial`)
- Semantic colors — no hardcoded hex
- SF Symbols — no custom icons
- System typography (`font(.headline)`, `font(.caption)`, etc.)
- Native spacing grid (4 pt base)
- Correct `LSUIElement` setup (no Dock icon)
- Optional `designlang` workflow for extracting design tokens from an HTML reference

## Installation

### 1. Clone

```bash
git clone https://github.com/timlimlei/native-macos-menubar.git
cd native-macos-menubar
```

### 2. Copy skill to Claude Code

```bash
cp -r . ~/.claude/skills/native-macos-menubar
```

Or symlink so updates are live:

```bash
ln -sf "$(pwd)" ~/.claude/skills/native-macos-menubar
```

### 3. Restart Claude Code

```bash
claude
```

The skill registers automatically on startup. Confirm with:

```
/skills
```

You should see `native-macos-menubar` in the list.

## Usage

In any Claude Code session, trigger the skill:

```
/native-menubar
```

Or reference it inline:

```
Use the /native-menubar skill to build a menu bar app that shows system stats.
```

## What the skill covers

| Section | Topics |
|---|---|
| Architecture | `MenuBarExtra` vs `NSStatusItem`, `.window` vs `.menu` style |
| Bootstrap | `@main`, `LSUIElement`, AppKit fallback for macOS 12 |
| Layout | Popover dimensions, row anatomy, spacing grid |
| SF Symbols | Rendering modes, status bar icon rules, symbol variants |
| Native controls | Borderless buttons, segmented pickers, toggles, search fields |
| Design tokens | `designlang` HTML-reference workflow → DTCG tokens → SwiftUI |
| State management | `@MainActor ObservableObject`, popover close patterns |
| Accessibility | Labels, values, hints |
| Common mistakes | Hardcoded colors, custom fonts, Dock icon appearing, wrong popover width |

## Requirements

- macOS 13+ for `MenuBarExtra` (SwiftUI)
- macOS 12 supported via AppKit fallback path in the skill
- Xcode 15+
- Node.js (only if using the `designlang` token-extraction workflow)

## designlang workflow (optional)

When building to match a specific existing app:

```bash
# 1. Build an HTML reference of your target UI
# 2. Serve it locally
python3 -m http.server 5173 --directory reference

# 3. Extract design tokens
npx designlang "http://localhost:5173/menubar.html" \
  --full --dark --selector "#app" \
  --name my-app -o ./designlang/my-app

# 4. Reference the output in your Claude prompt
# designlang/my-app/my-app-design-language.md
```

See `SKILL.md` for the complete workflow.

## License

MIT
