# mm-macos: Agentic Coder Implementation Plan

## How to Use This Plan

This document is a step-by-step build order for an agentic coder. Each phase produces a runnable increment. Follow phases in order. Within a phase, follow steps in order. Each step lists the files to create/modify and a verification method.

**Before starting:** Read `MM-MAC-PROJECT.md` for architecture context.

**Working directory:** All paths are relative to the project root. Create an Xcode project first (Phase 0), then implement files inside it.

**Testing rule:** After every step, run `xcodebuild test` (or `xcodebuild build` if no test target yet). Never proceed to the next step if the current one doesn't compile.

---

## Phase 0: Project Scaffold

### Step 0.1 — Create Xcode Project

Create a Swift AppKit project:

```
mm-macos/
├── mm-macos.xcodeproj
├── mm-macos/
│   ├── AppDelegate.swift
│   └── Info.plist
└── mm-macosTests/
    └── mm_macosTests.swift
```

**Actions:**
1. Create the Xcode project with `xcodebuild` or via Xcode GUI (Swift, AppKit, Storyboard-free)
2. Set deployment target to macOS 11.0
3. Remove the default `ViewController.swift` and `Main.storyboard` if created
4. Set `NSPrincipalClass` in Info.plist to use programmatic window creation
5. Configure `AppDelegate` to create a plain `NSWindow` with `NSViewController`

**Verify:** App builds and launches showing an empty window.

### Step 0.2 — Add Core/ UI/ Services/ Groups

Add folder groups in the Xcode project matching the architecture:

```
mm-macos/
├── Core/
├── UI/
└── Services/
```

**Verify:** Folders exist in project navigator.

---

## Phase 1: Core Data Model (No UI)

Goal: All tree operations, serialization, and layout logic working and tested — no AppKit dependency.

### Step 1.1 — MindmapNode.swift

**File:** `Core/MindmapNode.swift`

Port from C# `MindmapNode.cs` (16 lines). Changes:
- `Guid` → `UUID`
- `RectangleF` → `CGRect`
- `List<MindmapNode>` → `[MindmapNode]`
- `NodeSide` enum → `Core/NodeSide.swift`

```swift
enum NodeSide: String, Codable {
    case auto, left, right
}

class MindmapNode {
    let id: UUID
    var text: String = ""
    weak var parent: MindmapNode?
    var children: [MindmapNode] = []
    var bounds: CGRect = .zero
    var side: NodeSide = .auto

    init(id: UUID = UUID()) {
        self.id = id
    }
}
```

**Verify:** `xcodebuild build` compiles. Write unit test: create node, set properties, verify.

### Step 1.2 — Connection.swift

**File:** `Core/Connection.swift`

Port from C# `Connection.cs` (11 lines). Straightforward:

```swift
class Connection {
    let id: UUID
    var source: MindmapNode
    var target: MindmapNode

    init(id: UUID = UUID(), source: MindmapNode, target: MindmapNode) {
        self.id = id
        self.source = source
        self.target = target
    }
}
```

**Verify:** Compiles. Unit test: create connection between two nodes.

### Step 1.3 — Mindmap.swift

**File:** `Core/Mindmap.swift`

Port from C# `Mindmap.cs` (124 lines). This is the core tree model. Key methods:
- `addChild(_:text:)` — with left/right auto-alternation for root children
- `addSibling(_:text:)`
- `removeNode(_:)` — reparent children to grandparent
- `removeNodeWithChildren(_:)`
- `addConnection(source:target:)` / `removeConnection(_:)`
- `allNodes` computed property (depth-first traversal)

**Verify:** Unit tests for:
- Add child → verify parent/child links
- Add sibling → verify insertion order
- Remove node → children reparented
- Remove node with children → subtree gone
- All nodes traversal count

### Step 1.4 — DslSerializer.swift

**File:** `Core/DslSerializer.swift`

Port from C# `DslSerializer.cs` (299 lines). Must produce **byte-identical output** to the C# version for the same input (format compatibility).

Key details:
- `format=1` header
- `style=mindmap` header (always, theme is app-level)
- `title=` header
- Short IDs (`N1`, `N2`, ...)
- Side tags `[Left]` / `[Right]`
- Free connections `-->`
- `*` prefix for selected node/connection
- `DslParseException` with line numbers
- Escape/unescape for labels (quotes, backslashes)

**Verify:** Unit tests ported from `Mindmap.Tests/DslSerializerTests.cs`:
- Round-trip: serialize → deserialize → serialize matches
- Parse error cases (unknown child ref, duplicate ID, unterminated string)
- Side preservation
- Connection serialization

### Step 1.5 — TextMeasurer Protocol + LayoutEngine

**Files:**
- `Core/TextMeasurer.swift` (protocol)
- `Core/LayoutEngine.swift` (port from C#)

**TextMeasurer protocol:**
```swift
struct FontDescriptor: Equatable {
    let name: String
    let size: CGFloat
    let bold: Bool
}

protocol TextMeasurer {
    func measure(_ text: String, font: FontDescriptor) -> CGSize
    func lineHeight(for font: FontDescriptor) -> CGFloat
}
```

**LayoutEngine** port from C# `LayoutEngine.cs` (147 lines). Changes:
- Takes `TextMeasurer` instead of `Graphics`
- `CGRect` instead of `RectangleF`
- `CGPoint` instead of `PointF`
- Extension methods on `CGRect` (centerX, centerY, etc.)
- Constants: `HGap=60`, `VGap=12`, `PadX=16`, `PadY=8`, `MinWidth=80`
- Root font: system bold 18pt, node font: system 14pt
- Split root children by `NodeSide` (left/right/auto-alternate)
- Position root at center (0,0)
- Layout subtrees recursively

**Verify:** Unit test with a mock `TextMeasurer` that returns fixed sizes:
- Single root node → centered at origin
- Root + 2 children → one left, one right
- Root + 4 children → alternating sides
- Deep tree → correct vertical spacing

### Step 1.6 — Theme.swift

**File:** `Core/Theme.swift`

Extract color constants from C# `Renderer.cs`. Two static themes:

```swift
struct Theme: Equatable {
    let name: String
    let bgColor: NSColor
    let nodeFill: NSColor
    let nodeFillSelected: NSColor
    let nodeBorder: NSColor
    let nodeBorderSelected: NSColor
    let textColor: NSColor
    let textSelected: NSColor
    let edgeColor: NSColor  // with alpha
    let edgeSelectedColor: NSColor
    let gridDotColor: NSColor?     // nil for blueprint (uses lines)
    let gridLineColor: NSColor?    // nil for dark (uses dots)
    let gridMajorColor: NSColor?   // nil for dark
    let cornerRadius: CGFloat      // 16
    let gridSpacing: CGFloat       // 24

    static let blueprint: Theme  // blue background, grid lines
    static let dark: Theme       // Catppuccin Mocha, dot grid
}
```

Use exact color values from the C# source. Note: `Theme` uses `NSColor` so it's in Core but references AppKit. If you want Core to be truly platform-free, use a generic `Color` protocol — but for MVP, `NSColor` is fine.

**Verify:** Compiles. `Theme.blueprint` and `Theme.dark` exist with correct values.

### Step 1.7 — UndoManager (Command Pattern)

**File:** `Core/UndoManager.swift`

Replace the C# Memento approach with a Command pattern:

```swift
protocol UndoCommand {
    func undo(on map: Mindmap)
    func redo(on map: Mindmap)
}

class UndoManager {
    private var undoStack: [UndoCommand] = []
    private var redoStack: [UndoCommand] = []
    private let maxHistory = 100

    func execute(_ command: UndoCommand, on map: Mindmap) {
        command.redo(on: map)
        undoStack.append(command)
        redoStack.removeAll()
        trimStack()
    }

    func undo(on map: Mindmap) -> Bool { ... }
    func redo(on map: Mindmap) -> Bool { ... }
    func clear() { ... }
}
```

**Command implementations (create as needed in later phases):**
- `InsertNodeCommand` — stores parentId, nodeId, text, side, index
- `DeleteNodeCommand` — stores node snapshot (text, children, side, position) for restore
- `EditTextCommand` — stores nodeId, oldText, newText
- `MoveNodeCommand` — stores nodeId, oldParent, oldIndex, newParent, newIndex

**For MVP text-edit coalescing:** Add `executeCoalescing(_ command: EditTextCommand, on map: Mindmap)` that merges with the previous command if same node and within 500ms.

**Verify:** Unit tests:
- Execute command → undo → state restored
- Execute 3 commands → undo 3 times → original state
- Redo after undo
- Stack capped at 100
- Text edit coalescing within 500ms

### Step 1.8 — Logger.swift

**File:** `Core/Logger.swift`

Replace C# file logger with `os.log`:

```swift
import os

struct Logger {
    private static let logger = Logger(subsystem: "com.mm.macos", category: "general")

    static func info(_ message: String) { logger.info("\(message)") }
    static func warn(_ message: String) { logger.warning("\(message)") }
    static func error(_ message: String) { logger.error("\(message)") }
}
```

**Verify:** Compiles.

---

## Phase 2: AppKit Rendering (Visible but Static)

Goal: A window that displays a mindmap. No interaction yet.

### Step 2.1 — AppKitTextMeasurer

**File:** `UI/AppKitTextMeasurer.swift`

Implements `TextMeasurer` protocol using `NSFont` and `NSString.size(with:)`:

```swift
class AppKitTextMeasurer: TextMeasurer {
    func measure(_ text: String, font: FontDescriptor) -> CGSize {
        let nsFont = font.bold
            ? NSFont.boldSystemFont(ofSize: font.size)
            : NSFont.systemFont(ofSize: font.size)
        let lineCount = text.components(separatedBy: "\n").count
        let lineHeight = nsFont.boundingRectForFont.height
        let attributes: [NSAttributedString.Key: Any] = [.font: nsFont]
        let size = (text as NSString).size(withAttributes: attributes)
        return CGSize(width: max(size.width, 80), height: lineHeight * CGFloat(lineCount) + 16)
    }

    func lineHeight(for font: FontDescriptor) -> CGFloat {
        let nsFont = font.bold
            ? NSFont.boldSystemFont(ofSize: font.size)
            : NSFont.systemFont(ofSize: font.size)
        return nsFont.boundingRectForFont.height
    }
}
```

**Verify:** Unit test: measure "Hello" → returns non-zero CGSize.

### Step 2.2 — Renderer.swift

**File:** `UI/Renderer.swift`

Port drawing logic from C# `Renderer.cs` (323 lines). Key changes:
- `NSGraphicsContext` instead of `Graphics`
- `NSBezierPath` instead of `GraphicsPath`
- `NSColor` instead of `Color`
- `Theme` struct instead of hardcoded color properties
- Clean signature: `draw(context: NSGraphicsContext, map: Mindmap, viewport: Viewport, theme: Theme, highlights: Highlights)`

Where:
```swift
struct Viewport {
    let offsetX: CGFloat
    let offsetY: CGFloat
    let scale: CGFloat
    let visibleRect: CGRect
}

struct Highlights {
    let selectedNode: MindmapNode?
    let editingNode: MindmapNode?
    let dragNode: MindmapNode?
    let dropTarget: MindmapNode?
    let dropType: DropType
    let selectedConnection: Connection?
    let connectionSource: MindmapNode?
}
```

Drawing methods to port:
- `drawGrid` — blueprint: lines + major lines; dark: dots
- `drawEdges` — bezier curves from parent to child
- `drawConnections` — bezier curves for free connections
- `drawNodes` — rounded rect + centered text
- `drawDropIndicator` — insertion line for drag-drop (defer to post-MVP)

**Verify:** Build + manual test: create a `Mindmap` with 3 nodes, run layout, render in a window. Should show root + 2 children with bezier edges.

### Step 2.3 — MindmapView.swift

**File:** `UI/MindmapView.swift`

Custom `NSView` subclass:
- Overrides `draw(_ dirtyRect:)` to call `Renderer.draw()`
- Stores `map: Mindmap`, `theme: Theme`, `viewport: Viewport`
- Calls `setNeedsDisplay(_:)` when state changes
- Uses `NSView.layerContentsRedrawPolicy = .onMainThread` for efficiency

**Verify:** Window shows a static mindmap with correct theme colors.

### Step 2.4 — MainViewController.swift (Initial)

**File:** `UI/MainViewController.swift`

Minimal `NSViewController`:
- Creates `MindmapView` as the main view
- Sets up a sample `Mindmap` with "Central Idea" root + 2 children
- Runs `LayoutEngine.Layout()` with `AppKitTextMeasurer`
- Passes `map`, `theme`, `viewport` to `MindmapView`

**Verify:** App launches, shows a mindmap with 3 nodes. No interaction yet.

---

## Phase 3: Keyboard Navigation + Editing

Goal: Full zero-mode editing — the core UX.

### Step 3.1 — MindmapViewModel.swift

**File:** `UI/MindmapViewModel.swift`

The central orchestrator. Extracted from C# `MainForm.cs` logic:

```swift
class MindmapViewModel {
    let map: Mindmap
    let undoManager: UndoManager
    let layoutEngine: LayoutEngine
    let textMeasurer: TextMeasurer

    var selectedNode: MindmapNode? { map.selectedNode }
    var isEditing: Bool = false

    // Called by InputHandler after each mutation:
    func insertChild(text: String = "")
    func insertSibling(text: String = "")
    func deleteNode()
    func navigateUp() / navigateDown() / navigateLeft() / navigateRight()
    func startEditing()
    func commitEditing(newText: String)
    func cancelEditing()
    func undo() -> Bool
    func redo() -> Bool

    // Notifies view to refresh:
    var onMapChanged: (() -> Void)?
}
```

Navigation logic (port from C# `MainForm.cs` lines handling arrow keys):
- **Up/Down:** Navigate between siblings (same parent)
- **Left:** Go to parent (or first child if coming from right)
- **Right:** Go to first child (or next sibling if leaf)
- **Tab:** Same as INS (add child)
- **Enter:** Add sibling below
- **Backspace:** Delete last char; if empty, delete node

**Verify:** Unit tests for each navigation case (root, leaf, middle of tree).

### Step 3.2 — InputHandler.swift

**File:** `UI/InputHandler.swift`

Translates `NSEvent` (keyDown, mouseDown, etc.) into `MindmapViewModel` calls:

```swift
class InputHandler {
    weak var viewModel: MindmapViewModel?

    func handleKeyDown(_ event: NSEvent) -> Bool {
        let modifiers = event.modifierFlags.intersection(.deviceIndependentFlagsMask)
        switch (event.keyCode, modifiers) {
        case (36, []):  // Enter → insert sibling
        case (126, []): // Up → navigate up
        // ... etc
        }
    }

    func handleMouseDown(_ event: NSEvent, in view: NSView) -> Bool { ... }
    func handleMouseDragged(_ event: NSEvent, in view: NSView) -> Bool { ... }
    func handleScrollWheel(_ event: NSEvent, in view: NSView) -> Bool { ... }
}
```

Key mappings (from C# → macOS):
| C# Key | macOS Key | Action |
|--------|-----------|--------|
| INS | Tab / Cmd+N | Add child |
| Enter | Enter | Add sibling |
| Arrow keys | Arrow keys | Navigate |
| Backspace | Delete | Delete char / node |
| Ctrl+Z | Cmd+Z | Undo |
| Ctrl+Y | Cmd+Shift+Z | Redo |
| Ctrl+S | Cmd+S | Save |
| Ctrl+O | Cmd+O | Open |
| Ctrl+F | Cmd+F | Search (post-MVP) |
| Delete | Fn+Delete | Delete node |

**Verify:** Manual test: navigate between nodes, add children, add siblings, delete.

### Step 3.3 — EditOverlay.swift

**File:** `UI/EditOverlay.swift`

In-place text editing using `NSTextView` (not `NSTextField` — we need multiline):

```swift
class EditOverlay: NSTextView {
    var onCommit: ((String) -> Void)?
    var onCancel: (() -> Void)?

    // Position over the selected node
    func position(over node: MindmapNode, in view: MindmapView, viewport: Viewport) { ... }

    // Handle Enter (commit), Escape (cancel), Alt+Enter (newline)
    override func keyDown(_ event: NSEvent) { ... }
}
```

Port the C# `TextBox` overlay logic from `MainForm.cs`:
- Position overlay exactly over the node bounds (adjusted for viewport offset/scale)
- Commit on Enter, cancel on Escape
- Alt+Enter inserts newline
- Live-update node text as user types (TextChanged → update node → re-layout → keep node center fixed on screen)

**Verify:** Click a node → type → text appears in node → Enter commits → Escape cancels.

### Step 3.4 — Wire Up Editing in MainViewController

**File:** `UI/MainViewController.swift` (update)

Connect `MindmapViewModel` → `InputHandler` → `MindmapView` + `EditOverlay`:

```swift
class MainViewController: NSViewController {
    let viewModel = MindmapViewModel(...)
    let inputHandler = InputHandler()
    let mindmapView = MindmapView()
    let editOverlay = EditOverlay()

    override func viewDidLoad() {
        viewModel.onMapChanged = { [weak self] in
            self?.mindmapView.refresh()
        }
        inputHandler.viewModel = viewModel
    }
}
```

**Verify:** Full editing cycle: click node, type, see text update, add child, navigate, delete.

---

## Phase 4: Pan, Zoom, and Viewport

Goal: Trackpad/mouse pan and zoom.

### Step 4.1 — Pan (Scroll Wheel / Two-Finger Drag)

**File:** `UI/MindmapView.swift` (update)

Handle `scrollWheel(with:)` events:
- Plain scroll → pan (update `offsetX`, `offsetY`)
- Cmd+scroll → zoom (update `scale`)
- Clamp scale between 0.2 and 5.0

Port the C# pan/zoom logic from `MainForm.cs`:
- `_offsetX`, `_offsetY`, `_scale` state
- Mouse drag for panning (middle button or space+drag)
- Zoom centers on cursor position (not viewport center)

**Verify:** Two-finger scroll pans the view. Cmd+scroll zooms. Zoom centers on cursor.

### Step 4.2 — Mouse Click Selection

**File:** `UI/InputHandler.swift` (update)

Handle `mouseDown` in `MindmapView`:
- Hit-test: find node under cursor (iterate `map.allNodes`, check if point is in `node.bounds`)
- If hit → select that node
- If miss → deselect
- Double-click → start editing

**Verify:** Click a node → it highlights. Click empty space → deselect. Double-click → edit.

---

## Phase 5: File I/O

Goal: Open and save `.mm` files. Format-compatible with the C# version.

### Step 5.1 — FileService.swift

**File:** `Services/FileService.swift`

Port from C# `FileService.cs` (119 lines). Changes:
- `Environment.GetFolderPath(SpecialFolder.ApplicationData)` → `~/Library/Application Support/mm/`
- `File.ReadAllText` / `File.WriteAllText` → Swift `String(contentsOf:)` / `try text.write(toFile:)`
- Settings: `UserDefaults` instead of JSON file
- Error handling: Swift `Result` type or `throws`

```swift
class FileService {
    func loadMindmap(from path: String) throws -> Mindmap { ... }
    func saveMindmap(_ map: Mindmap, to path: String) throws { ... }
    func loadTheme() -> String? { UserDefaults.standard.string(forKey: "theme") }
    func saveTheme(_ theme: String) { UserDefaults.standard.set(theme, forKey: "theme") }
}
```

**Verify:** Unit test: save a `Mindmap` → load it back → compare. Test with a `.mm` file created by the C# version.

### Step 5.2 — Open/Save Dialogs

**File:** `UI/MainViewController.swift` (update)

Add `Cmd+O` (open) and `Cmd+S` (save) handling:
- `NSOpenPanel` for open
- `NSSavePanel` for save
- Track `_currentFile` path and `_isDirty` flag
- Mark dirty on any mutation
- Reset dirty after save
- On window close: prompt to save if dirty (port C# `OnFormClosing` logic)

**Verify:** Cmd+O opens a `.mm` file. Cmd+S saves. Closing dirty file prompts save dialog.

### Step 5.3 — Drag-and-Drop File Open

**File:** `UI/MainViewController.swift` (update)

Register for file drag-and-drop:
- Accept `.mm` files
- On drop → load file

**Verify:** Drag a `.mm` file onto the window → it opens.

---

## Phase 6: Undo/Redo Integration

Goal: Full undo/redo with Cmd+Z / Cmd+Shift+Z.

### Step 6.1 — Wire UndoManager into ViewModel

**File:** `UI/MindmapViewModel.swift` (update)

Every mutation method must create an `UndoCommand` and call `undoManager.execute()`:

```swift
func insertChild(text: String = "") {
    let command = InsertNodeCommand(parentId: selectedNode.id, text: text, side: .auto)
    undoManager.execute(command, on: map)
    layoutEngine.layout(map: map, measurer: textMeasurer)
    onMapChanged?()
}
```

**Verify:** Add 3 nodes → undo 3 times → all gone → redo 3 times → all back.

### Step 6.2 — Text Edit Coalescing

**File:** `Core/UndoManager.swift` (update)

Add `executeCoalescing` for text edits:
- If the last command is an `EditTextCommand` for the same node and < 500ms elapsed, merge the new text into it
- Otherwise, push a new command

**Verify:** Type 10 characters → undo once → all 10 revert. Wait 1 second → type more → undo → only last batch reverts.

---

## Phase 7: Theme Switching + Context Menu

Goal: Switch between blueprint and dark themes. Right-click context menu.

### Step 7.1 — Theme Switching

**File:** `UI/MainViewController.swift` (update)

- Store current theme as `Theme.blueprint` or `Theme.dark`
- Context menu items: "Blueprint theme" / "Dark theme"
- On switch: update `mindmapView.theme`, save to `UserDefaults`
- Port C# `DarkMenuRenderer` for styled context menu (or use `NSMenu` with `NSAppearance`)

**Verify:** Right-click → switch theme → view updates. Restart app → theme persists.

### Step 7.2 — Context Menu

**File:** `UI/MainViewController.swift` (update)

Port context menu items from C# `MainForm.cs`:
- Blueprint theme / Dark theme
- Save / Save As... / Open...
- Export... (post-MVP)
- Full Screen

**Verify:** Right-click shows context menu. Items work.

---

## Phase 8: Polish + MVP Completion

Goal: Ship the MVP.

### Step 8.1 — Window Title + Dirty Indicator

**File:** `UI/MainViewController.swift` (update)

- Title: "mm - {filename}" or "mm - untitled"
- Dirty indicator: append "•" or "*" when `_isDirty`
- Port C# `UpdateTitle()` logic

**Verify:** Title updates on open, save, edit.

### Step 8.2 — Auto-Save (Crash Recovery)

**File:** `Services/FileService.swift` (update)

- Every 60 seconds, if dirty, save to `~/Library/Application Support/mm/autosave.mm`
- On launch, check for autosave file → offer to restore
- Delete autosave on successful save or explicit close

**Verify:** Edit → force quit → relaunch → prompted to recover.

### Step 8.3 — Full Screen

**File:** `UI/MainViewController.swift` (update)

- Toggle full screen with `NSWindow.toggleFullScreen(_:)`
- Port C# `ToggleFullScreen()` logic
- Save/restore window frame

**Verify:** Ctrl+Cmd+F toggles full screen. Escape exits.

### Step 8.4 — App Icon + Dock

**Files:** `Assets.xcassets/AppIcon.appiconset/`

- Create or port the app icon from C# `Packaging/Assets/app.ico`
- Add to asset catalog

**Verify:** App shows icon in Dock.

---

## Post-MVP Features (Not in Initial Build)

These are documented for future reference. Do NOT implement until the MVP is complete and stable.

| Feature | Phase | Notes |
|---------|-------|-------|
| Free connections (Ctrl+Click → drag) | P9 | Port from C# `MainForm.cs` connection mode |
| Drag-drop reordering | P9 | Port C# drag-drop logic |
| PNG export | P10 | Port `PngExporter.cs` using `NSBitmapImageRep` |
| HTML export | P10 | Port `HtmlExporter.cs` + `SvgBuilder.cs` |
| Search (Cmd+F) | P11 | Port C# search bar logic |
| Symbol popup | P11 | Port `SymbolPopup.cs` as `NSPopover` |
| Incremental layout (dirty flags) | P12 | Performance optimization for large maps |
| Drag-drop from Finder | P12 | Already have file drag-drop; add to dock icon |

---

## Testing Strategy

### Unit Tests (mm-macosTests target)

| Component | What to Test |
|-----------|-------------|
| `MindmapNode` | Creation, property mutation |
| `Mindmap` | AddChild, AddSibling, RemoveNode, RemoveNodeWithChildren, AllNodes, Connections |
| `DslSerializer` | Round-trip, parse errors, side preservation, connections, format version |
| `LayoutEngine` | Single root, 2 children, 4 children alternating, deep tree, mock TextMeasurer |
| `UndoManager` | Execute/undo/redo, stack cap, text coalescing |
| `FileService` | Save/load round-trip, C# format compatibility |

### Manual Test Checklist (Run After Each Phase)

- [ ] App launches, shows empty mindmap with "Central Idea"
- [ ] Tab adds child, Enter adds sibling
- [ ] Arrow keys navigate between nodes
- [ ] Delete removes last character; on empty node, removes node
- [ ] Typing updates node text in real-time
- [ ] Cmd+Z undoes, Cmd+Shift+Z redoes
- [ ] Two-finger scroll pans, Cmd+scroll zooms
- [ ] Click selects node, double-click edits
- [ ] Cmd+O opens .mm file, Cmd+S saves
- [ ] Theme switch works and persists
- [ ] Window title shows filename and dirty indicator

---

## Build Commands Reference

```bash
# Build
xcodebuild -project mm-macos.xcodeproj -scheme mm-macos build

# Run tests
xcodebuild -project mm-macos.xcodeproj -scheme mm-macos test

# Run app
open -a mm-macos
```

---

## File Creation Order Summary

| Order | File | Phase | Depends On |
|-------|------|-------|------------|
| 1 | `Core/NodeSide.swift` | 1.1 | — |
| 2 | `Core/MindmapNode.swift` | 1.1 | NodeSide |
| 3 | `Core/Connection.swift` | 1.2 | MindmapNode |
| 4 | `Core/Mindmap.swift` | 1.3 | MindmapNode, Connection |
| 5 | `Core/DslSerializer.swift` | 1.4 | Mindmap, MindmapNode, Connection, NodeSide |
| 6 | `Core/TextMeasurer.swift` | 1.5 | — |
| 7 | `Core/LayoutEngine.swift` | 1.5 | MindmapNode, Mindmap, TextMeasurer |
| 8 | `Core/Theme.swift` | 1.6 | — |
| 9 | `Core/UndoManager.swift` | 1.7 | Mindmap |
| 10 | `Core/Logger.swift` | 1.8 | — |
| 11 | `UI/AppKitTextMeasurer.swift` | 2.1 | TextMeasurer |
| 12 | `UI/Renderer.swift` | 2.2 | Theme, Mindmap, Connection |
| 13 | `UI/MindmapView.swift` | 2.3 | Renderer, Theme |
| 14 | `UI/MainViewController.swift` | 2.4 | MindmapView, Mindmap, LayoutEngine |
| 15 | `UI/MindmapViewModel.swift` | 3.1 | Mindmap, UndoManager, LayoutEngine |
| 16 | `UI/InputHandler.swift` | 3.2 | MindmapViewModel |
| 17 | `UI/EditOverlay.swift` | 3.3 | MindmapNode |
| 18 | `Services/FileService.swift` | 5.1 | DslSerializer |
| 19 | `AppDelegate.swift` | 0.1 | MainViewController |
