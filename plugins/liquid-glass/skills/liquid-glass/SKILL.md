---
name: liquid-glass
description: Build and migrate iOS, macOS, iPadOS, watchOS, tvOS, and visionOS apps with Apple's Liquid Glass design system (iOS 26+, macOS 26 Tahoe+). Use when creating new SwiftUI apps targeting Apple's 2025+ platforms, migrating existing apps to Liquid Glass, applying .glassEffect(), .backgroundExtensionEffect(), .buttonStyle(.glass), GlassEffectContainer, glass toolbars, glass tab bars, glass sidebars, glass sheets, concentric corners, safeAreaBar, or any Liquid Glass UI work. Also use for graceful degradation and backward compatibility when deployment targets are below iOS 26, Liquid Glass app icons and Icon Composer layers, UIDesignRequiresCompatibility, modern Apple design, navigation patterns with glass, or SwiftUI best practices for the latest OS versions.
---

# Liquid Glass Design System for Apple Platforms

Build and migrate SwiftUI apps using Apple's Liquid Glass design language introduced at WWDC 2025. This skill covers iOS 26, iPadOS 26, macOS 26 (Tahoe), watchOS 26, tvOS 26, and visionOS 26.

**visionOS exception:** SwiftUI's glass APIs (`glassEffect`, `Glass`, `GlassEffectContainer`, `glassEffectID`, `.buttonStyle(.glass)`) are marked unavailable on visionOS. There, custom glass comes from `glassBackgroundEffect`; `backgroundExtensionEffect` is available.

## Important: Use Latest Documentation

Always check current Apple documentation and the SDK/compiler used by the project before implementing. Availability and details can vary by platform and SDK. Key references:

- [Adopting Liquid Glass](https://developer.apple.com/documentation/technologyoverviews/adopting-liquid-glass)
- [Materials HIG](https://developer.apple.com/design/human-interface-guidelines/materials)
- [Applying Liquid Glass to custom views](https://developer.apple.com/documentation/SwiftUI/Applying-Liquid-Glass-to-custom-views)
- [`glassEffect(_:in:)`](https://developer.apple.com/documentation/swiftui/view/glasseffect%28_%3Ain%3A%29)
- [Landmarks: Building an app with Liquid Glass](https://developer.apple.com/documentation/SwiftUI/Landmarks-Building-an-app-with-Liquid-Glass)
- `https://developer.apple.com/videos/play/wwdc2025/323/` (WWDC25 Session: Build a SwiftUI app with the new design)

## Core Concept

Liquid Glass is a dynamic material for the **functional layer** (navigation and controls) that floats above app content. System components adopt it automatically. Prefer those components; add custom glass sparingly to important controls. Standard materials remain appropriate for backgrounds and structure in the content layer. Liquid Glass is not exclusive to navigation: transient controls embedded in content can use it to emphasize interactivity.

Avoid applying Liquid Glass to content itself (lists, tables, media, text blocks). Use standard materials for content surfaces. Use `.clear` only over visually rich media; when the content beneath is bright, HIG suggests a dark dimming layer at 35% opacity. If the content beneath is already dark — or you use AVKit playback controls, which dim themselves — no dimming layer is needed. Keep foreground labels legible and use semantic/vibrant foreground colors.

## Quick Start: Key APIs

### 1. Glass Effect on Custom Views

```swift
// Basic - capsule shape (default)
Text("Label")
    .padding()
    .glassEffect()

// With shape and style
Image(systemName: "heart.fill")
    .padding()
    .glassEffect(.regular, in: .rect(cornerRadius: 16))

// Tinted glass
Text("Tinted")
    .padding()
    .glassEffect(.regular.tint(.blue))

// Interactive glass - visual feedback on interaction.
// Available on macOS too (AppKit's effectIsInteractive is API_AVAILABLE(macos(27.0))),
// but documented for glass backing interactive controls: on a glass surface hosting
// only static text, macOS shows no response to hover or press.
Button("Tap Me") { }
    .glassEffect(.regular.interactive())
```

### 2. Glass Styles

| Style | Use Case | Transparency |
|-------|----------|-------------|
| `.regular` | Standard UI: toolbars, buttons, nav bars | Medium |
| `.clear` | Media-rich backgrounds where content is bold/bright | High |
| `.identity` | Conditionally disable glass (accessibility) | None |

### 3. GlassEffectContainer (Critical)

A container merges nearby glass elements and batches their rendering. AppKit documents
`NSGlassEffectContainerView` as a view that "efficiently merges descendant glass effect views together
when they are within a specified proximity to each other," with a performance tip about "reducing the
number of passes required to render similar glass effect views." It is also what makes morphing work.

Use `GlassEffectContainer` for multiple effects that should blend or morph, and when coordinating them
with `glassEffectID`, `glassEffectUnion`, or `glassEffectTransition`. `spacing` sets how close shapes must
be before they blend: a larger value blends sooner, and a value larger than the interior layout spacing
makes effects blend even at rest. Avoid excessive containers or onscreen effects; profile complex interfaces.

```swift
GlassEffectContainer(spacing: 30.0) {
    Button("Action 1") { }
        .glassEffect()
        .glassEffectID("btn1", in: namespace)

    Button("Action 2") { }
        .glassEffect()
        .glassEffectID("btn2", in: namespace)
}
```

### 4. Morphing Transitions

Use `@Namespace` + `glassEffectID` inside a `GlassEffectContainer` for smooth glass morphing:

```swift
@Namespace private var namespace
@State private var isExpanded = false

GlassEffectContainer(spacing: 16) {
    if isExpanded {
        ForEach(items) { item in
            ItemView(item: item)
                .glassEffect(.regular, in: .rect(cornerRadius: 24))
                .glassEffectID(item.id, in: namespace)
        }
    }
    Button {
        withAnimation { isExpanded.toggle() }
    } label: { Label("Toggle", systemImage: "chevron.down") }
    .buttonStyle(.glass)
    .glassEffectID("toggle", in: namespace)
}
```

`matchedGeometry` is the default transition for effects that appear or disappear within the container's
spacing; use `.materialize` for effects farther apart. With the default animation, `matchedGeometry` also
adds scale and offset to content when a shape's identity stays the same but its content changes — pass an
explicit `.spring` to opt out of those extra animations.

### 5. Background Extension Effect

Extends and blurs visual content behind navigation elements (toolbars, sidebars, inspectors):

```swift
Image(landmark.backgroundImageName)
    .resizable()
    .aspectRatio(contentMode: .fill)
    .frame(minWidth: 0, maxWidth: .infinity, minHeight: 0, maxHeight: .infinity)
    .backgroundExtensionEffect()
```

Align the view's edges with the containing view (no padding on the enclosing `ScrollView`/`VStack`);
the effect only mirrors into available safe-area space. Apply the modifier before adding overlays, so
only the image extends under the sidebar or inspector and the labels stay put.

### 6. Glass Buttons

```swift
// Standard glass button
Button("Action") { }
    .buttonStyle(.glass)

// Prominent glass button (for primary actions)
Button("Save") { }
    .buttonStyle(.glassProminent)

// Configurable variant: pass any Glass, e.g. clear over media
Button("Play") { }
    .buttonStyle(.glass(.clear))
```

### 7. Toolbars with Glass

Toolbar items automatically receive glass styling. Use `ToolbarSpacer` and `ToolbarItemGroup` for layout:

```swift
.toolbar {
    ToolbarSpacer(.flexible)
    ToolbarItem { ShareLink(item: data, preview: preview) }
    ToolbarSpacer(.fixed)
    ToolbarItemGroup {
        Button("Favorite", systemImage: "heart") { }
        Button("Add", systemImage: "plus") { }
    }
    ToolbarItem {
        Button("Info", systemImage: "info") { }
    }
}
.toolbar(removing: .title) // Remove title for clean glass toolbar
```

Group items that affect the same part of the interface, and don't mix text and icon items inside one glass
group. Give every icon an accessibility label. To hide an item, hide the `ToolbarItem` itself
(`ToolbarContent.hidden(_:)`, or `isHidden` on `UIBarButtonItem`/`NSToolbarItem`) rather than emptying its
content, which leaves a blank glass pill. `ToolbarSpacer` is available on iOS, iPadOS, Mac Catalyst, and macOS.

For a custom bar that should get the same legibility treatment as a system toolbar, use
`.safeAreaBar(edge: .bottom) { ... }`: it insets the content, adjusts the safe area, and shapes the scroll
edge effect around the bar.

### 8. Tab Bars

Tab bars automatically adopt glass when compiled with Xcode 26:

```swift
TabView {
    Tab("Home", systemImage: "house") { HomeView() }
    Tab("Search", systemImage: "magnifyingglass") { SearchView() }
}
.tabBarMinimizeBehavior(.onScrollDown) // Collapse tab bar on scroll (iOS only)
```

`.onScrollDown`, `.onScrollUp`, and `.never` are iOS-only values; `.automatic` is the portable one.

Use `.tabViewStyle(.sidebarAdaptable)` to let a tab bar become a sidebar on iPad and Mac (UIKit:
`UITabBarController.Mode.tabSidebar`). Give the search tab `role: .search`, so the system separates it and
places it at the trailing end. `.tabViewBottomAccessory` is iOS/iPadOS/Catalyst only: it sits above the tab
bar at normal size and inline when the bar is minimized.

### 9. Sheets with Glass

Partial-height sheets automatically get Liquid Glass backgrounds:

```swift
.sheet(isPresented: $showSheet) {
    SheetContent()
        .presentationDetents([.medium, .large])
    // Do NOT add .presentationBackground() - system handles it
}
```

Half sheets are inset from the display edge (so content peeks through) and use a larger corner radius; at
full height the sheet turns more opaque to keep focus on the task. Don't add a visual-effect view behind
sheet or popover content — remove existing ones so they match the rest of the system.

## Migration Workflow (Existing Apps)

For detailed migration steps, see [references/migration-guide.md](references/migration-guide.md).

**Summary:**
0. Optional: add `UIDesignRequiresCompatibility` = `YES` to ship with the new SDK while keeping the old UI
   during review. It is temporary — the system ignores it once you build with the 27-era SDKs.
1. Compile with Xcode 26 SDK - system components auto-adopt glass
2. Remove custom toolbar backgrounds (`.toolbarBackground`) where they conflict with system glass
3. Replace custom materials on navigation elements with glass modifiers
4. Wrap grouped glass elements in `GlassEffectContainer`
5. Add `.backgroundExtensionEffect()` to hero images
6. Drop `.circle` symbol variants: toolbar buttons are no longer circled on iOS 26+, so use plain symbols with `.symbolVariant()` for state
7. Rebuild the app icon in Icon Composer with separate layers and check the light, dark, clear, and tinted variants
8. Test accessibility (Reduced Transparency, Increased Contrast, Reduced Motion, preferred Liquid Glass look)

Deploying below the 26 OSes? Availability is the risk, not the design. See
[references/graceful-degradation.md](references/graceful-degradation.md) for the platform-complete `#available`
rule, the visionOS `#if` split, and a fallback for every 26-only API.

## Platform Considerations

For platform-specific details, see [references/platform-specifics.md](references/platform-specifics.md).

**Key differences:**
- **macOS**: Use `.tint(.clear)` on glass buttons for proper rendering; use `WindowBackgroundShapeStyle.windowBackground` instead of `Material` for editing backgrounds; `NSButton.BezelStyle.glass` styles AppKit buttons
- **iOS (iPhone)**: `.interactive()` works; use `UIDevice.current.userInterfaceIdiom` for layout
- **iPadOS**: Larger grid sizes; sidebar adaptable tab views (`.tabViewStyle(.sidebarAdaptable)`); windows resize continuously, so support arbitrary sizes
- **watchOS**: Changes are minimal and appear automatically even without rebuilding; adopt standard toolbar APIs and button styles from watchOS 10
- **tvOS**: Liquid Glass requires Apple TV 4K (2nd generation) or newer — older devices keep their current appearance. Standard focus APIs drive the effect; on tvOS `.buttonStyle(.glass)` applies glass whether or not the button has focus
- **Conditional compilation**: Use `#if os(macOS)` / `#if os(iOS)` for platform-specific code
- **UIKit / AppKit**: Native glass APIs exist (`UIGlassEffect`, `UIGlassContainerEffect`, `NSGlassEffectView`, and containers) when the interface isn't SwiftUI
- **visionOS**: The SwiftUI glass family is unavailable; use `glassBackgroundEffect` (visionOS 1+) for custom glass, plus `backgroundExtensionEffect` / `UIBackgroundExtensionView`, which do ship on visionOS

## Common Pitfalls

For detailed pitfalls and solutions, see [references/pitfalls-and-solutions.md](references/pitfalls-and-solutions.md).

**Critical issues:**
- Glass elements outside `GlassEffectContainer` produce inconsistent visuals
- `rotationEffect` on glass views can cause shape morphing artifacts - validate on target OS versions before choosing a UIKit workaround
- Menu labels with glass cause animation artifacts - use custom `ButtonStyle`
- Hit-testing only registers on content, not glass area - use `contentShape()` to fix
- Multiple glass effects = multiple `CABackdropLayer` instances (3 offscreen textures each) - use containers to group

## Real-World Example Patterns

For complete code patterns from Apple's Landmarks sample app, see [examples/landmarks-patterns.md](examples/landmarks-patterns.md).

## Architecture Best Practices

1. **NavigationSplitView** as app root with glass sidebar
2. **NavigationStack** for deep navigation within detail columns
3. **@Observable** data model with `@Environment` injection
4. **FlexibleHeader** pattern: stretching hero images with scroll-linked parallax
5. **.inspector()** for supplementary detail panels
6. **.searchable()** for global search (auto-styled with glass)
7. Use `ToolbarSpacer(.flexible)` and `ToolbarSpacer(.fixed)` for toolbar layout
8. Use icon buttons for common actions and give each one an accessibility label; don't mix text and icon items in one glass group
9. Use `.symbolVariant()` modifier for SF Symbol state changes
10. Use `.symbolEffect(.drawOn)` for animated icon transitions
11. Use `glassEffectTransition(.matchedGeometry)` or `.materialize` to select how custom glass enters/leaves a hierarchy
12. Tune scroll/content boundaries with `scrollEdgeEffectStyle(_:for:)` and `scrollEdgeEffectHidden(_:for:)` when automatic behavior does not fit
13. Match device and container curvature with `ConcentricRectangle`, or `.rect(corners: .concentric(minimum:))` backed by `.containerShape(...)`. `containerConcentric` is a masking style, not a corner radius — `.rect(cornerRadius: .containerConcentric)` does not compile
14. Prefer standard `List`/`Form` styles (e.g. `.formStyle(.grouped)`): larger row heights, rounder sections, and title-style section headers arrive automatically. Section headers are no longer uppercased, so write them in title case
15. Anchor action sheets and confirmation dialogs to the control that triggered them (`sourceView` / `sourceItem`), rather than assuming they rise from the bottom edge
