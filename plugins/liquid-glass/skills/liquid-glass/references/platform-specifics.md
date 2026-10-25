# Platform-Specific Liquid Glass Implementation

Detailed platform considerations for implementing Liquid Glass across Apple platforms.

## Table of Contents

1. [iOS 26 (iPhone)](#ios-26-iphone)
2. [iPadOS 26](#ipados-26)
3. [macOS 26 (Tahoe)](#macos-26-tahoe)
4. [watchOS 26](#watchos-26)
5. [tvOS 26](#tvos-26)
6. [visionOS 26](#visionos-26)
7. [Cross-Platform Patterns](#cross-platform-patterns)

---

## iOS 26 (iPhone)

### Glass Features Available
- Full `.glassEffect()` with all styles (`.regular`, `.clear`, `.identity`)
- `.interactive()` glass - scaling, bouncing, shimmering on touch
- `GlassEffectContainer` with morphing transitions
- `.backgroundExtensionEffect()` for hero images
- `.buttonStyle(.glass)` and `.buttonStyle(.glassProminent)`
- Automatic glass tab bars with `.tabBarMinimizeBehavior(.onScrollDown)`
- Glass sheets with `.presentationDetents([.medium, .large])`
- Glass search bars via `.searchable()`

### iPhone-Specific Patterns

```swift
#if os(iOS)
struct InspectorToolbar: ViewModifier {
    @Binding var inspectorIsPresented: Bool

    func body(content: Content) -> some View {
        content
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                if UIDevice.current.userInterfaceIdiom == .phone {
                    ToolbarItem {
                        Button { inspectorIsPresented.toggle() } label: {
                            Label("Toggle Inspector", systemImage: "info.circle")
                        }
                    }
                }
            }
    }
}

// Smaller grid item sizes for compact screens (inside a constants type)
var gridItemMinSize: CGFloat {
    UIDevice.current.userInterfaceIdiom == .phone ? 160.0 : 240.0
}
#endif
```

### Navigation on iPhone
- `NavigationSplitView` collapses to single column
- Use `preferredCompactColumn` to control default view
- Search bar dynamically moves based on scroll position

---

## iPadOS 26

### iPad-Specific Features
- Floating Liquid Glass sidebar
- Side-by-side split view with glass navigation
- Inspector panels with glass styling
- Larger touch targets for glass buttons

### iPad Layout Patterns

```swift
#if os(iOS)
// Inside a constants type
static var gridItemMinSize: CGFloat {
    UIDevice.current.userInterfaceIdiom == .pad ? 220.0 : 160.0
}

static var gridItemEditingMinSize: CGFloat {
    UIDevice.current.userInterfaceIdiom == .pad ? 180.0 : 160.0
}
#endif
```

### Tab bar to sidebar

```swift
TabView { /* tabs */ }
    .tabViewStyle(.sidebarAdaptable)
```

UIKit: `UITabBarController.Mode.tabSidebar`. The system decides tab bar versus sidebar from context, so
don't branch on size class to pick a layout.

### Windows and multitasking

iPadOS windows have window controls and resize continuously rather than snapping between presets. Support
arbitrary window sizes, let `NavigationSplitView` reflow its columns, and set safe areas so the system can
position window controls and the title bar around your content.

### Split View with Glass

```swift
NavigationSplitView(preferredCompactColumn: $preferredColumn) {
    // Sidebar - auto-glass on iPad
    List {
        ForEach(NavigationOptions.mainPages) { page in
            NavigationLink(value: page) {
                Label(page.name, systemImage: page.symbolName)
            }
        }
    }
    .frame(minWidth: 150)
} detail: {
    NavigationStack(path: $modelData.path) {
        MainContentView()
    }
}
.searchable(text: $modelData.searchString, prompt: "Search")
.inspector(isPresented: $modelData.isInspectorPresented) {
    InspectorView()
}
```

---

## macOS 26 (Tahoe)

### macOS-Specific Considerations

**Don't bridge `NSGlassEffectView` as a bare `.background`.** This is the most common way macOS glass
goes wrong, and it fails silently. `NSGlassEffectView` is documented as "a view that embeds its content
view in a dynamic glass effect," and its `contentView` property is **nullable** — so wrapping it in an
`NSViewRepresentable` and applying it as a SwiftUI `.background(...)` compiles, runs, and renders a flat
grey slab with a thick uniform rim. A `.background(...)` layers the glass view *behind* your content as a
sibling, so the content slot stays empty and the material has nothing to lens. No `cornerRadius` or
padding value will thin that rim, because it is a rendering artifact, not a border.

```swift
// WRONG - flat slab with a thick grey rim
.background(GlassBackground(radius: 24))   // NSViewRepresentable, contentView never set

// RIGHT
.glassEffect(.regular, in: .rect(cornerRadius: 24))
```

Prefer `.glassEffect(_:in:)` on macOS. The shape argument supplies the content relationship the bridged
view was missing, so the effect is fully specified by construction.

**Turn off the window shadow under glass.** On a borderless `NSPanel`/`NSWindow` whose content is a glass
surface, set `hasShadow = false`. Liquid Glass draws its own shadow shaped to the glass; AppKit's window
shadow is shaped to the window. Stacked, they read as one thick, wrong-shaped edge that looks like a
border.

```swift
panel.hasShadow = false   // .glassEffect draws its own
```

**Verifying glass visually:** a screenshot test is only valid if the backdrop's features are larger than
the blur radius. Fine stripes, small text, and dense noise all average to a flat grey field under the
backdrop blur, so a *working* effect looks broken and you conclude glass can't sample that content. Use
large, saturated colour blocks with a hard boundary running under the element, then check two things: the
boundary visible through the material (transmission), and the boundary **displaced** near the rounded
corners (refraction — this is the one that distinguishes real glass from a plain blur).

**Glass button tinting:** Always apply `.tint(.clear)` to glass buttons on macOS:

```swift
Button("Action") { }
    .buttonStyle(.glass)
    #if os(macOS)
    .tint(.clear)
    #endif
```

**Background styles:** Use `WindowBackgroundShapeStyle` instead of `Material`:

```swift
#if os(macOS)
static let editingBackgroundStyle = WindowBackgroundShapeStyle.windowBackground
.background(Color(nsColor: .windowBackgroundColor))
.background(Color(nsColor: isEditing ? .secondarySystemFill : .windowBackgroundColor))
#else
static let editingBackgroundStyle = Material.ultraThickMaterial
.background(Color(uiColor: .systemBackground))
#endif
```

**Toolbar placement:** Use `.secondaryAction` for overflow items:

```swift
#if os(macOS)
let deleteButtonPlacement: ToolbarItemPlacement = .secondaryAction
let editButtonPlacement: ToolbarItemPlacement = .automatic
#elseif os(iOS)
let deleteButtonPlacement: ToolbarItemPlacement = .topBarLeading
let editButtonPlacement: ToolbarItemPlacement = .topBarTrailing
#endif
```

**AppKit buttons:** `NSButton.BezelStyle.glass` gives an AppKit button the glass appearance without bridging
SwiftUI. `NSGlassEffectView` exposes `style`, `tintColor`, `cornerRadius`, `contentView`, and (macOS 27+)
`effectIsInteractive`.

**Window configuration:**

```swift
WindowGroup {
    ContentView()
        .frame(minWidth: 375.0, minHeight: 375.0)
}
```

### macOS Navigation
- `NavigationSplitView` shows persistent sidebar
- Window toolbar integrates glass automatically
- Inspector panels slide from trailing edge
- Search bar appears in toolbar area

---

## watchOS 26

### Glass on Apple Watch

Liquid Glass changes on watchOS are deliberately minimal, and they appear on the latest OS even if the app was
not rebuilt against the new SDK. What reliably picks them up is using standard button styles and toolbar APIs
(the watchOS 10 APIs) — not hand-rolling glass.

- Navigation and toolbars are handled by the system; don't rebuild them from custom glass.
- Prefer standard button styles. `.glassEffect` is available on watchOS 26, but screen area is scarce.
- Keep custom glass to one small, transient element, and profile it.
- Don't remove or replace the material backgrounds that modal sheets get by default; watchOS relies on them
  for contrast.

### Considerations

- With a container that blends shapes, a small screen leaves little room before everything reads as one blob.
- Test with Always-On display and with every complication/watch face you support.
- Reduce Motion and Reduce Transparency matter more here: the display is small and always with the person.

---

## tvOS 26

Liquid Glass requires Apple TV 4K (2nd generation) or newer; older devices keep their current appearance.

### Glass on Apple TV

- Standard buttons and controls take on glass when focus moves to them. Adopt the standard focus APIs so
  custom controls match (`focusable(_:)`, `@FocusState`, `isFocused`).
- In tvOS, `.buttonStyle(.glass)` and `.glassProminent` apply the effect **whether or not** the button has
  focus — so don't reach for them expecting focus-driven styling.
- `ToolbarSpacer` is not available on tvOS; size toolbar gaps with the platform's own toolbar APIs.
- `backgroundExtensionEffect()` is available and suits media-rich layouts.

### Focus-Aware Glass

```swift
@FocusState private var isFocused: Bool

Button("Play") { }
    .focusable()
    .focused($isFocused)
    // Let the system style the standard button on focus; use isFocused only for extra custom emphasis
```

---

## visionOS 26

### Glass in Spatial Computing

visionOS does **not** use SwiftUI's control-oriented glass APIs. `glassEffect`, `Glass`,
`GlassEffectContainer`, `glassEffectID`, `glassEffectUnion`, `glassEffectTransition`, and
`.buttonStyle(.glass)` are all marked unavailable there.

- Windows are drawn with a system-defined glass material you cannot modify. Prefer translucency over opaque
  fills: opacity blocks people's view of their surroundings and makes them feel constricted.
- For a custom glass surface, use `glassBackgroundEffect(displayMode:)` (visionOS 1+). `GlassBackgroundEffect`
  adds `.automatic`, `.plate`, and `.feathered` configurations (visionOS 2.4+).
- Ornaments, toolbars, and window chrome receive system glass automatically.
- `backgroundExtensionEffect()` and `UIBackgroundExtensionView` do exist on visionOS, unlike the rest of the
  glass family.
- visionOS applies vibrancy to text, symbols, and fills over materials; use the three visionOS vibrancy values
  to communicate hierarchy rather than hard-coded colors.

---

## Cross-Platform Patterns

### Conditional Compilation Pattern

```swift
#if os(macOS)
// macOS-specific implementation
#elseif os(iOS)
if UIDevice.current.userInterfaceIdiom == .pad {
    // iPad-specific
} else {
    // iPhone-specific
}
#elseif os(watchOS)
// Watch-specific
#elseif os(tvOS)
// TV-specific
#elseif os(visionOS)
// Vision-specific
#endif
```

### Shared Architecture Pattern

Use a single codebase with platform adaptations:

```swift
@main
struct MyApp: App {
    @State private var modelData = ModelData()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(modelData)
                .frame(minWidth: 375.0, minHeight: 375.0)
                .onGeometryChange(for: CGSize.self) { geometry in
                    geometry.size
                } action: {
                    modelData.windowSize = $0
                }
        }
    }
}
```

### Responsive Layout Constants

```swift
struct LayoutConstants {
    @MainActor static var gridItemMinSize: CGFloat {
        #if os(iOS)
        if UIDevice.current.userInterfaceIdiom == .pad {
            return 220.0  // iPad
        } else {
            return 160.0  // iPhone
        }
        #elseif os(macOS)
        return 240.0
        #elseif os(watchOS)
        return 80.0
        #else
        return 200.0
        #endif
    }
}
```

### NavigationSplitView Across Platforms

```swift
NavigationSplitView(preferredCompactColumn: $preferredColumn) {
    // Sidebar: glass on all platforms
    SidebarContent()
} detail: {
    // Detail: NavigationStack for deep navigation
    NavigationStack(path: $path) {
        DetailContent()
    }
    .navigationDestination(for: Item.self) { item in
        ItemDetailView(item: item)
    }
}
// Global search - positioned by system per platform
.searchable(text: $searchString, prompt: "Search")
// Inspector - system handles presentation per platform
.inspector(isPresented: $showInspector) {
    InspectorContent()
}
```
