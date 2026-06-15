---
name: native-macos-menubar
description: "Build native macOS menu bar apps — the kind Apple ships for WiFi, Bluetooth, Battery, etc. Covers MenuBarExtra/NSStatusItem, system materials, SF Symbols, semantic colors, native popover layout, and the designlang HTML-reference workflow for extracting design tokens from a reference UI."
trigger: /native-menubar
---

# /menubar — Native macOS Menu Bar App

Build a native macOS menu bar app indistinguishable from Apple's own (WiFi, Bluetooth, Battery, Passwords).

## Mental Model

Apple menu bar extras come in two flavors:

| Flavor | Style | Use when |
|---|---|---|
| **Menu** | `NSMenu` dropdown | simple actions, no live state |
| **Window** | `NSPopover` / `NSWindow` | rich UI, search, lists |

Always prefer **`MenuBarExtra`** (SwiftUI, macOS 13+) over raw `NSStatusItem` unless you need macOS 12 support.

---

## Step 0 — Decide Architecture

```
App needs custom UI (list, search, forms)?
  → .menuBarExtraStyle(.window)

App needs simple menu items?
  → .menuBarExtraStyle(.menu)

Must support macOS 12?
  → NSStatusItem + NSPopover (AppKit)
```

---

## Step 1 — Bootstrap the App

### SwiftUI (macOS 13+, preferred)

```swift
import SwiftUI

@main
struct MyMenuBarApp: App {
    var body: some Scene {
        MenuBarExtra("MyApp", systemImage: "key.fill") {
            ContentView()
        }
        .menuBarExtraStyle(.window)  // or .menu
    }
}
```

**Remove `WindowGroup`** — menu bar apps must not show a dock icon or main window unless explicitly triggered.

`Info.plist` — add:
```xml
<key>LSUIElement</key>
<true/>
```

### AppKit fallback (macOS 12)

```swift
class AppDelegate: NSObject, NSApplicationDelegate {
    var statusItem: NSStatusItem!
    var popover = NSPopover()

    func applicationDidFinishLaunching(_ notification: Notification) {
        statusItem = NSStatusBar.system.statusItem(withLength: NSStatusItem.variableLength)
        if let button = statusItem.button {
            button.image = NSImage(systemSymbolName: "key.fill", accessibilityDescription: nil)
            button.action = #selector(togglePopover)
        }
        popover.contentViewController = NSHostingController(rootView: ContentView())
        popover.behavior = .transient
    }

    @objc func togglePopover() {
        guard let button = statusItem.button else { return }
        if popover.isShown {
            popover.performClose(nil)
        } else {
            popover.show(relativeTo: button.bounds, of: button, preferredEdge: .minY)
        }
    }
}
```

---

## Step 2 — Native Layout Primitives

Use exactly what Apple uses. Nothing custom unless unavoidable.

### Material backgrounds

```swift
// .window style MenuBarExtra auto-applies material.
// For manual NSPopover or sub-panels:
.background(.regularMaterial)           // standard popover (WiFi, Bluetooth)
.background(.ultraThinMaterial)         // lightweight overlay
.background(.thickMaterial)             // more opaque (Control Center)
```

### Semantic colors — ALWAYS use system colors, never hardcode hex

```swift
.foregroundStyle(.primary)             // primary text
.foregroundStyle(.secondary)           // subtitles, captions
.foregroundStyle(.tertiary)            // placeholder, disabled
Color.accentColor                       // tint/interactive elements
Color(nsColor: .controlBackgroundColor) // row backgrounds
Color(nsColor: .separatorColor)         // dividers
```

### Typography — system text styles only

```swift
.font(.title2)          // popover header
.font(.headline)        // row title
.font(.subheadline)     // row subtitle
.font(.caption)         // metadata, timestamps
.font(.footnote)        // legal, hint text
// Never hardcode size+weight unless absolutely required.
// Use .fontDesign(.default) — never set family to anything other than system.
```

### Spacing grid

```swift
// Apple uses 4pt base grid.
4    // micro gap (icon + label)
8    // between related elements
12   // section padding
16   // section separation
20   // major section gap
```

### Row anatomy (matches Bluetooth / WiFi rows)

```swift
struct NativeRow: View {
    let icon: String     // SF Symbol name
    let title: String
    let subtitle: String?

    var body: some View {
        HStack(spacing: 8) {
            Image(systemName: icon)
                .symbolRenderingMode(.hierarchical)
                .frame(width: 20, alignment: .center)
                .foregroundStyle(.secondary)

            VStack(alignment: .leading, spacing: 1) {
                Text(title)
                    .font(.headline)
                if let subtitle {
                    Text(subtitle)
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }

            Spacer()
        }
        .padding(.horizontal, 12)
        .frame(height: 44)
        .contentShape(Rectangle())
    }
}
```

### Popover dimensions (match Apple standard)

```swift
.frame(width: 320)          // compact (Battery, Clock)
.frame(width: 360)          // standard (WiFi, Bluetooth, Passwords)
.frame(width: 400)          // wide (Control Center panels)
// Height: let content drive height, set a .frame(maxHeight: 480) cap.
```

---

## Step 3 — SF Symbols (mandatory, never custom icons)

```swift
Image(systemName: "wifi")
Image(systemName: "key.fill")
Image(systemName: "lock.fill")
Image(systemName: "ellipsis.circle")
Image(systemName: "plus.circle")

// Rendering modes:
.symbolRenderingMode(.monochrome)      // menu bar status icon
.symbolRenderingMode(.hierarchical)    // in-popover icons (depth)
.symbolRenderingMode(.multicolor)      // system-colored (Battery etc.)
.symbolVariant(.fill)                  // filled variant
.imageScale(.small / .medium / .large)
```

**Status bar icon rules:**
- Always monochrome (template image)
- Weight: `.medium` or `.regular` — never `.heavy`
- Use `Image(systemName:)` set as `statusItem.button.image` via `NSImage(systemSymbolName:)`
- Template images auto-invert for light/dark menu bars

---

## Step 4 — Native Controls

Use AppKit-mapped SwiftUI controls. No custom buttons.

```swift
// Borderless button (all Apple popover actions use this)
Button("Open App") { }
    .buttonStyle(.borderless)
    .foregroundStyle(Color.accentColor)

// Segmented control
Picker("Mode", selection: $mode) {
    Text("A").tag("a")
    Text("B").tag("b")
}
.pickerStyle(.segmented)

// Toggle (WiFi on/off style)
Toggle("WiFi", isOn: $enabled)
    .toggleStyle(.switch)

// Search field (Passwords, Spotlight-adjacent)
TextField("Search", text: $query)
    .textFieldStyle(.roundedBorder)

// Divider
Divider()
    .padding(.vertical, 4)

// Section header (Bluetooth "My Devices" style)
Text("DEVICES")
    .font(.caption2)
    .fontWeight(.semibold)
    .foregroundStyle(.secondary)
    .padding(.horizontal, 12)
    .padding(.top, 8)
```

---

## Step 5 — Design Token Extraction (designlang workflow)

When building to match a specific existing app or design reference:

### 5a. Build HTML reference

Create `reference/menubar.html` modeling the target UI with CSS variables:

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <style>
    :root {
      --popover-width: 360px;
      --popover-radius: 16px;
      --popover-padding: 12px;
      --row-height: 44px;
      --control-height: 30px;
      --spacing-xs: 4px;
      --spacing-sm: 8px;
      --spacing-md: 12px;
      --spacing-lg: 16px;
      --text-primary: rgba(0,0,0,0.88);
      --text-secondary: rgba(0,0,0,0.55);
      --separator: rgba(0,0,0,0.12);
      --surface: rgba(246,246,246,0.82);
      --surface-elevated: rgba(255,255,255,0.88);
      --accent: #007aff;
      --font-family: -apple-system, BlinkMacSystemFont, "SF Pro Text", system-ui, sans-serif;
    }
    @media (prefers-color-scheme: dark) {
      :root {
        --text-primary: rgba(255,255,255,0.92);
        --text-secondary: rgba(255,255,255,0.58);
        --separator: rgba(255,255,255,0.14);
        --surface: rgba(36,36,38,0.82);
        --surface-elevated: rgba(44,44,46,0.88);
      }
    }
    body { margin: 0; min-height: 100vh; display: grid; place-items: center;
           background: #8e8e93; font-family: var(--font-family); }
    .popover {
      width: var(--popover-width);
      border-radius: var(--popover-radius);
      padding: var(--popover-padding);
      background: var(--surface);
      box-shadow: 0 18px 60px rgba(0,0,0,0.28);
      backdrop-filter: blur(28px) saturate(1.6);
      color: var(--text-primary);
    }
  </style>
</head>
<body>
  <main id="app" class="popover" data-component="menubar-popover">
    <!-- Model your UI states here as data-component annotated elements -->
  </main>
</body>
</html>
```

### 5b. Serve and extract

```bash
python3 -m http.server 5173 --directory reference

npx designlang "http://localhost:5173/menubar.html" \
  --full \
  --dark \
  --selector "#app" \
  --name my-menubar-app \
  -o ./designlang/my-menubar-app
```

### 5c. Key outputs

```
designlang/my-menubar-app/
  my-menubar-app-design-language.md   ← curate this, use as Claude prompt context
  my-menubar-app-design-tokens.json   ← DTCG tokens
  my-menubar-app-variables.css        ← CSS custom properties
  my-menubar-app-anatomy.tsx          ← component anatomy
```

### 5d. Reference the design language in your implementation prompt

```
Use ./designlang/my-menubar-app/my-menubar-app-design-language.md.
Implement a native SwiftUI macOS MenuBarExtra app.
Use native macOS materials, semantic colors, SF Symbols, system typography.
Do not clone Apple assets or proprietary UI exactly.
```

---

## Step 6 — State Management Patterns

### Global app state (accessible from menu bar and any opened windows)

```swift
@MainActor
class AppState: ObservableObject {
    @Published var isLocked = true
    @Published var items: [Item] = []
    static let shared = AppState()
}

// In App:
MenuBarExtra(...) {
    ContentView().environmentObject(AppState.shared)
}
```

### Closing the popover from inside content

```swift
// For .window style MenuBarExtra
struct ContentView: View {
    @Environment(\.openWindow) var openWindow
    // No direct close API — user clicks outside (transient behavior)
    // To force close: post notification or use NSApp
}

// AppKit approach
NSApp.sendAction(#selector(NSPopover.performClose(_:)), to: nil, from: nil)
```

---

## Step 7 — Accessibility

```swift
// Every interactive element needs a label
Image(systemName: "wifi")
    .accessibilityLabel("WiFi")

Button("...") { }
    .accessibilityLabel("More options")

// Dynamic state
.accessibilityValue(isEnabled ? "On" : "Off")
.accessibilityHint("Double-tap to toggle")
```

---

## Project Structure

```
MyMenuBarApp/
  App/
    MyMenuBarApp.swift          // @main, MenuBarExtra scene
    AppDelegate.swift           // AppKit fallback if needed
  Views/
    ContentView.swift           // root popover view
    RowViews.swift              // reusable row components
    DetailView.swift            // drill-down state
    EmptyStateView.swift
    LockedStateView.swift
  Models/
    AppState.swift              // @MainActor ObservableObject
    Item.swift
  designlang/                   // extracted design tokens (if used)
  reference/                    // HTML reference pages (if used)
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Hardcoded hex colors | Use semantic colors: `.primary`, `.secondary`, `Color.accentColor` |
| Custom fonts | `font(.headline)` — system only |
| PNG icons | SF Symbols via `Image(systemName:)` |
| Dock icon shows | `LSUIElement = true` in Info.plist |
| Popover too wide | 320–400pt max |
| Fixed height rows | `frame(height: 44)` + `contentShape(Rectangle())` |
| `.sheet` / `.window` from menu bar | Use `NSWorkspace.shared.open(url)` or `openWindow` env |
| Background not translucent | `.background(.regularMaterial)` on root VStack |
| SwiftUI `WindowGroup` present | Remove it; MenuBarExtra is the only scene |
