# mm-macos Project Summary

## Chosen Approach: Swift + AppKit

For the macOS port of **mm**, we selected **Swift + AppKit** over alternatives:

| Option | Why Not Chosen |
|--------|----------------|
| .NET MAUI | macOS support less mature; rendering differences |
| Avalonia UI | Cross-platform but different input model; learning curve |
| Swift + SwiftUI | Requires macOS 10.15+; steeper learning curve for complex custom drawing |

**Why AppKit:**
- Native performance and full control over rendering
- Direct access to low-level drawing APIs (`NSBezierPath`, `NSGraphicsContext`)
- Easiest deployment (no runtime dependencies)
- Perfect fit for keyboard-first, zero-mode editing UX

---

## Lessons from the Original .NET Project

The original `mm` (C# / WinForms) was studied in full before designing this port. Key findings:

### What Worked Well (Port 1:1)
- **DSL format** — versioned, compact, round-trip safe, platform-agnostic
- **Tree model** — `MindmapNode`, `Mindmap`, `Connection` are clean and simple
- **Layout algorithm** — directional split (left/right children of root), subtree height caching
- **FileService** — clean separation of I/O from logic
- **Two themes** — dark (Catppuccin Mocha) and blueprint, easily extensible

### What Needs Redesign
| Problem | Original | macOS Fix |
|---------|----------|-----------|
| MainForm is a 1881-line god object | All input, editing, search, file ops, undo triggers in one class | Split into ViewModel + InputHandler + SearchController |
| Undo uses full-tree Memento cloning | `MindmapCloner.Capture()` deep-clones entire tree per snapshot | Command pattern — O(1) per undo entry |
| LayoutEngine depends on GDI+ `Graphics` | `MeasureString()` requires a WinForms graphics context | `TextMeasurer` protocol — no `NSGraphicsContext` dependency |
| Renderer takes 8 parameters | `Draw(g, map, offsetX, offsetY, scale, dragNode, dropTarget, dropType, visibleArea, editingNode)` | Clean `draw(context, map, viewport, highlights)` with `Theme` struct injected |
| No incremental layout | Every keystroke remeasures all nodes | Dirty-flag on nodes; only remeasure changed nodes and recompute ancestor subtree heights |
| UI logic untested | All interaction logic in MainForm, no unit tests | ViewModel extraction makes logic testable from day one |

---

## Prerequisites

### Hardware
- macOS 11.0+ (Big Sur) or later
- 4GB RAM minimum, 8GB recommended
- Trackpad or mouse with scroll wheel support

### Software

#### Xcode (Required)
**Where to get:**
- [Mac App Store](https://apps.apple.com/app/xcode/id497799835) — Recommended, automatic updates
- [Apple Developer Downloads](https://developer.apple.com/download/all/) — For beta/older versions (requires free Apple ID)

**Version:** Xcode 15.0+ (includes Swift 5.9+, macOS SDK 14.0+)

**Installation:**
```bash
# Via App Store (GUI)
open -a "Mac App Store" --args itms-92826372

# Or via command line (requires Command Line Tools first)
sudo xcode-select --install
xcode-install  # if using xcode-install tool
```

#### Swift (Included with Xcode)
Swift is bundled with Xcode. To use system-wide:
```bash
# Add Xcode Swift to PATH (add to ~/.zshrc or ~/.bashrc)
export PATH="/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin:$PATH"
```

#### macOS SDK
Included with Xcode. Target SDK version: 14.0 (Sonoma)

### Development Environment Setup

```bash
# Install Xcode Command Line Tools (if not installing full Xcode)
xcode-select --install

# Verify installation
xcodebuild -version
swift --version

# Accept Xcode license (required before first build)
sudo xcodebuild -license accept
```

---

## Project Structure

```
mm-macos/
├── Core/                          # Platform-agnostic, testable
│   ├── MindmapNode.swift          # Data model (UUID, NSRect) — port 1:1 from C#
│   ├── Mindmap.swift              # Tree operations + connections — port 1:1
│   ├── Connection.swift           # Free connections between nodes — port 1:1
│   ├── NodeSide.swift            # Auto/Left/Right enum — port 1:1
│   ├── LayoutEngine.swift         # Directional layout — decoupled from Graphics
│   ├── TextMeasurer.swift         # PROTOCOL — abstracts text measurement
│   ├── UndoManager.swift          # Command-based, not Memento
│   ├── MindmapState.swift         # Snapshot for file serialization only
│   ├── DslSerializer.swift        # DSL format — port 1:1 (format-compatible)
│   ├── Theme.swift                # Extracted from Renderer (colors, fonts, radii)
│   └── Logger.swift               # os.log based (unified logging)
├── UI/                             # AppKit layer
│   ├── MainViewController.swift   # Thin coordinator — delegates to ViewModel
│   ├── MindmapView.swift          # Custom NSView — rendering only
│   ├── MindmapViewModel.swift     # State orchestration, undo triggers, editing lifecycle
│   ├── InputHandler.swift         # Key/mouse/gesture → command dispatch
│   ├── SearchController.swift     # Search bar + result navigation
│   ├── Renderer.swift             # Takes Theme + Canvas protocol
│   ├── SymbolPopup.swift          # Emoji/symbol picker — port from C#
│   └── EditOverlay.swift          # NSTextView overlay for in-place editing
├── Services/                       # File I/O, export
│   ├── FileService.swift           # Load/save .mm, settings via UserDefaults
│   ├── PngExporter.swift           # Uses Renderer + Canvas protocol
│   ├── HtmlExporter.swift          # Self-contained HTML export — port from C#
│   └── SvgBuilder.swift            # SVG edge/connection generation
└── mm-macos.xcodeproj
```

---

## Key Technical Decisions

| Decision | Rationale |
|----------|-----------|
| Keep core logic in Swift without UI dependencies | Enables unit testing; eases future cross-platform port |
| Use `NSBezierPath` for all drawing | Native macOS curve rendering with anti-aliasing |
| Map Ctrl → Cmd for shortcuts | Standard macOS convention (Cmd+S, Cmd+Z, etc.) |
| Trackpad two-finger scroll for pan/zoom | Natural macOS gesture; no middle mouse button |
| **ViewModel layer** between Core and UI | Extracted from MainForm god object; makes all interaction logic testable |
| **Command pattern** for undo | O(1) per undo entry vs O(n) full-tree Memento clone |
| **TextMeasurer protocol** for layout | Decouples LayoutEngine from NSGraphicsContext; enables headless testing |
| **Theme struct** injected into Renderer | Easy to add themes; no static color constants in renderer |
| **Dirty-flag layout** | Only remeasure changed nodes; recompute ancestor subtree heights |
| **DSL format unchanged** | File compatibility with the .NET version |

---

## Command Pattern for Undo

Instead of cloning the entire tree on each mutation (the original approach), each action stores its inverse:

```swift
protocol UndoCommand {
    func undo(map: Mindmap)
    func redo(map: Mindmap)
}

class InsertNodeCommand: UndoCommand {
    let parentId: UUID
    let nodeId: UUID
    let text: String
    let side: NodeSide
    let index: Int

    func undo(map: Mindmap) { /* remove node */ }
    func redo(map: Mindmap) { /* re-insert node */ }
}

class EditTextCommand: UndoCommand {
    let nodeId: UUID
    let oldText: String
    let newText: String

    func undo(map: Mindmap) { /* restore oldText */ }
    func redo(map: Mindmap) { /* apply newText */ }
}
```

Benefits:
- O(1) memory per undo entry (vs O(n) for Memento)
- Easy to add macro recording later
- Each command is independently testable

For the MVP, `MindmapState` snapshots can still be used as a fallback for complex operations (drag-drop reorder, connection changes).

---

## TextMeasurer Protocol

Decouples layout from platform rendering:

```swift
protocol TextMeasurer {
    func measure(_ text: String, font: FontDescriptor) -> CGSize
}

struct FontDescriptor {
    let name: String
    let size: CGFloat
    let bold: Bool
}
```

AppKit implementation:
```swift
class AppKitTextMeasurer: TextMeasurer {
    func measure(_ text: String, font: FontDescriptor) -> CGSize {
        let nsFont = font.bold
            ? NSFont.boldSystemFont(ofSize: font.size)
            : NSFont.systemFont(ofSize: font.size)
        // ... NSLayoutManager / NSString.size(with:) measurement
    }
}
```

This allows `LayoutEngine` to run in unit tests with a mock measurer, and enables PNG/HTML export without creating a live graphics context.

---

## Theme Struct

Extracted from the original's static color constants:

```swift
struct Theme {
    let name: String
    let bgColor: NSColor
    let nodeFill: NSColor
    let nodeFillSelected: NSColor
    let nodeBorder: NSColor
    let nodeBorderSelected: NSColor
    let textColor: NSColor
    let textSelected: NSColor
    let edgeColor: NSColor
    let edgeSelectedColor: NSColor
    let gridDotColor: NSColor?
    let gridLineColor: NSColor?
    let gridMajorColor: NSColor?
    let cornerRadius: CGFloat
    let gridSpacing: CGFloat

    static let blueprint = Theme(name: "blueprint", /* ... */)
    static let dark = Theme(name: "dark", /* Catppuccin Mocha colors */)
}
```

---

## First Working Version Scope

**MVP Features:**
- [ ] Root node displays on screen
- [ ] Type to edit (zero-mode editing)
- [ ] INS adds child, ENTER adds sibling
- [ ] Arrow keys navigate between nodes
- [ ] Backspace deletes last char / node if empty
- [ ] Layout updates as you type
- [ ] Cmd+Z / Cmd+Shift+Z undo/redo
- [ ] Cmd+S save, Cmd+O open .mm files
- [ ] Two themes: blueprint and dark

**Deferred (Post-MVP):**
- Free connections (Ctrl+Click → drag)
- Drag-drop reordering
- PNG export
- HTML export
- Full-screen mode
- Symbol popup
- Search (Ctrl+F)
- Incremental (dirty-flag) layout

---

## macOS Conventions Mapping

| Original (.NET) | macOS Port | Notes |
|-----------------|-----------|-------|
| `Ctrl+` shortcuts | `Cmd+` shortcuts | Standard macOS convention |
| `System.Drawing.Font` | `NSFont` | SF Pro / system font |
| `Graphics.MeasureString()` | `TextMeasurer` protocol | Decoupled for testability |
| `Graphics.DrawBezier()` | `NSBezierPath` | AppKit native curves |
| `TextBox` overlay | `NSTextView` overlay | In-place editing |
| `ContextMenuStrip` | `NSMenu` | Right-click context menu |
| `%AppData%/mm/` | `~/Library/Application Support/mm/` | macOS data directory |
| `Registry` / `settings.json` | `UserDefaults` | macOS preferences |
| `File.WriteAllText()` | `String.write(toFile:)` | Swift FileManager |
| `Environment.TickCount` | `ProcessInfo.processInfo.systemUptime` | Timing |
| `Form.DoubleBuffered` | `NSView.layerContentsRedrawPolicy` | AppKit buffering |
| `DragDropEffects` | `NSPasteboard` + `NSDragging` | macOS drag-and-drop |
| `Form.FormBorderStyle` | `NSWindow.styleMask` | Full-screen toggle |
</arg_value>`
**EOF:`MM-MAC-PROJECT.md**
