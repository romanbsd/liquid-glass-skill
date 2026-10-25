# Liquid Glass API Reference

API reference for Liquid Glass across SwiftUI, UIKit, and AppKit. Verify platform availability against
the active SDK. Apple references: [SwiftUI custom glass](https://developer.apple.com/documentation/SwiftUI/Applying-Liquid-Glass-to-custom-views),
[UIKit appearance APIs](https://developer.apple.com/documentation/uikit/appearance-customization),
[AppKit glass views](https://developer.apple.com/documentation/appkit/nsglasseffectview), and
[SwiftUI updates](https://developer.apple.com/documentation/updates/swiftui).

## Table of Contents

1. [glassEffect Modifier](#glasseffect-modifier)
2. [Glass Struct](#glass-struct)
3. [GlassEffectContainer](#glasseffectcontainer)
4. [glassEffectID](#glasseffectid)
5. [glassEffectUnion](#glasseffectunion)
6. [glassEffectTransition](#glasseffecttransition)
7. [backgroundExtensionEffect and scroll edges](#backgroundextensioneffect-and-scroll-edges)
8. [Button Styles](#button-styles)
9. [Toolbar Glass APIs](#toolbar-glass-apis)
10. [TabView Glass APIs](#tabview-glass-apis)
11. [Sheet Glass APIs](#sheet-glass-apis)
12. [visionOS Glass Background APIs](#visionos-glass-background-apis)
13. [UIKit and AppKit APIs](#uikit-and-appkit-apis)
14. [Accessibility APIs](#accessibility-apis)

---

## glassEffect Modifier

Applies the Liquid Glass material to a view.

```swift
func glassEffect(
    _ glass: Glass = .regular,
    in shape: some Shape = DefaultGlassEffectShape()
) -> some View
```

**Parameters:**
- `glass`: The glass material style (`.regular`, `.clear`, `.identity`)
- `shape`: The shape of the glass effect background

**Supported shapes:**
- `.capsule` (default)
- `.circle`
- `RoundedRectangle(cornerRadius: 16)` / `.rect(cornerRadius: 16)`
- `ConcentricRectangle` / `.rect(corners: .concentric(minimum: 16))` — corners concentric with the container shape
- `.ellipse`
- Any custom `Shape` conformance

**Examples:**

```swift
// Default capsule shape
Text("Glass").padding().glassEffect()

// Rounded rectangle
view.glassEffect(.regular, in: .rect(cornerRadius: 24))

// Circle
view.glassEffect(.regular, in: .circle)

// Concentric corners (matches the container)
view.glassEffect(.regular, in: ConcentricRectangle())
    // or: .glassEffect(.regular, in: .rect(corners: .concentric(minimum: 16)))

// Conditionally enabled
view.glassEffect(showGlass ? .regular : .identity)
```

`ConcentricRectangle` resolves each corner against the container shape. System views supply one; for custom
containers, declare it with `.containerShape(...)`. Note that `containerConcentric` belongs to
`CornerMaskingConfiguration`, not to `CGFloat` — `.rect(cornerRadius: .containerConcentric)` does not compile.

The currently documented SwiftUI signature has no `isEnabled` parameter. Apply this modifier after
other modifiers that affect appearance because SwiftUI captures the modified content.

**Availability:** iOS 26.0+, iPadOS 26.0+, Mac Catalyst 26.0+, macOS 26.0+, tvOS 26.0+, watchOS 26.0+.
**Unavailable on visionOS** — use `glassBackgroundEffect` there instead.

---

## Glass Struct

The `Glass` struct defines the material appearance.

### Static Properties

| Property | Description |
|----------|-------------|
| `.regular` | Standard glass: medium transparency, full adaptation. For toolbars, buttons, nav bars |
| `.clear` | High transparency, limited adaptation. For media-rich backgrounds with bold/bright overlay content |
| `.identity` | No glass effect. For conditionally disabling glass |

Availability matches `glassEffect`: iOS 26, iPadOS 26, Mac Catalyst 26, macOS 26, tvOS 26, watchOS 26 —
`Glass` is unavailable on visionOS.

### Instance Methods

```swift
// Add a tint color
func tint(_ color: Color?) -> Glass

// Enable interactive behavior (scaling, bouncing, shimmering on touch)
func interactive(_ isEnabled: Bool = true) -> Glass
```

**Method chaining:** Order-independent.

```swift
.glassEffect(.regular.tint(.blue).interactive())
// same as
.glassEffect(.regular.interactive().tint(.blue))
```

**Tint examples:**

```swift
.glassEffect(.regular.tint(.blue))
.glassEffect(.regular.tint(.purple.opacity(0.6)))
.glassEffect(.regular.tint(Color("accentColor")))
```

**Interactive behavior:**
- Scales on press
- Bouncing animation on release
- Shimmering effect
- Touch-point illumination radiating to nearby glass elements
- Tap and drag responsiveness

These are the touch-driven behaviors, so they are iOS-centric in practice. The modifier is **not**
unavailable on macOS, though: AppKit's equivalent flag is `effectIsInteractive`, declared
`API_AVAILABLE(macos(27.0))`, and documented for glass "used as the background for interactive controls
or when used as the container of interactive controls." On a macOS glass surface hosting only static
text, no response to hover or press is observable. macOS has no touch input and no accelerometer, so the
motion-driven specular response seen on iOS has no input source there — macOS glass responds to what
moves behind it.

---

## GlassEffectContainer

Groups multiple glass elements into a unified composition, merging them when they are in proximity and
batching their render passes. AppKit's `NSGlassEffectContainerView` documentation describes exactly this:
it "efficiently merges descendant glass effect views together when they are within a specified proximity
to each other" and "improve[s] performance by reducing the number of passes required to render similar
glass effect views." It is also the prerequisite for morphing.

A container is intended for multiple glass effects that should blend or morph, and improves rendering
performance. A single isolated glass view generally does not need one.

```swift
struct GlassEffectContainer<Content: View>: View {
    init(spacing: CGFloat? = nil, @ViewBuilder content: () -> Content)
    init(@ViewBuilder content: () -> Content)
}
```

**Parameters:**
- `spacing`: Controls proximity for blending and morphing. If larger than the spacing in an interior
  layout, shapes can blend while at rest.

Use containers when applying effects to multiple views to improve rendering performance and enable
shapes to blend and morph. Avoid relying on undocumented rendering implementation details.

```swift
GlassEffectContainer(spacing: 30.0) {
    // All glass elements share the same sampling region
    Button("A") { }.glassEffect()
    Button("B") { }.glassEffect()
    Button("C") { }.glassEffect()
}
```

---

## glassEffectID

Links glass elements for morphing transitions within a `GlassEffectContainer`.

```swift
func glassEffectID(
    _ id: (some Hashable & Sendable)?,
    in namespace: Namespace.ID
) -> some View
```

**Requirements for morphing:**
1. Elements must be in the same `GlassEffectContainer`
2. Each view needs `glassEffectID` with a shared `@Namespace`
3. Conditional show/hide triggers morphing animation
4. Wrap state change in `withAnimation`

```swift
@Namespace private var namespace
@State private var showExtra = false

GlassEffectContainer(spacing: 16) {
    Button("Toggle") {
        withAnimation(.bouncy) { showExtra.toggle() }
    }
    .glassEffect()
    .glassEffectID("toggle", in: namespace)

    if showExtra {
        Button("Extra") { }
            .glassEffect()
            .glassEffectID("extra", in: namespace)
    }
}
```

---

## glassEffectUnion

Combines multiple glass effects into a single unified glass shape.

```swift
func glassEffectUnion(
    id: (some Hashable & Sendable)?,
    namespace: Namespace.ID
) -> some View
```

All effects in the union need the same shape and `Glass` variant to combine. Use an optional stable
identifier scoped to a namespace.

---

## glassEffectTransition

Select how an effect enters or leaves the view hierarchy. Use with `glassEffect` and
`GlassEffectContainer`.

```swift
func glassEffectTransition(_ transition: GlassEffectTransition) -> some View
```

- `.matchedGeometry` derives the appearing shape from nearby glass in the same container.
- `.materialize` animates material and fades content without matching another shape; use when effects
  are farther apart than the container spacing.
- `.identity` applies no transition.

`matchedGeometry` is the default for effects that appear or disappear within the container's spacing. With
the default animation it also applies scale and offset to content when a shape's identity stays the same but
its content changes; pass an explicit `.spring` to opt out of those extra animations.

```swift
if isExpanded {
    Image(systemName: "note")
        .glassEffect()
        .glassEffectID("note", in: namespace)
        .glassEffectTransition(.materialize)
}
```

These modifiers affect view hierarchy transitions/animations. `withAnimation` controls the state change.

---

## backgroundExtensionEffect and scroll edges

Extends and blurs visual content beyond a view's bounds, creating continuous backgrounds behind navigation elements.

SwiftUI provides `backgroundExtensionEffect()` and `backgroundExtensionEffect(isEnabled: Bool)`. The
effect mirrors and blurs view content into available safe areas around its bounds. UIKit provides
`UIBackgroundExtensionView` for UIKit hierarchies.

Align the view's edges with the enclosing view — no padding on the surrounding `ScrollView`/`VStack`, and the
trailing edge flush with the containing view — because the effect only fills available safe-area space. Apply
it before adding overlays, so only the image extends beneath the sidebar or inspector.

**Availability:** iOS 26+, iPadOS 26+, Mac Catalyst 26+, macOS 26+, tvOS 26+, watchOS 26+, and visionOS 26+ —
one of the few glass modifiers that does ship on visionOS.

Apply it with discretion: the view is duplicated into mirrored, blurred copies, so prefer a single background
instance per layout and weigh visual clarity against the extra rendering.

**Use cases:**
- Hero images that extend behind toolbars/navigation bars
- Detail view images extending behind sidebar/inspector
- Featured content spanning multiple UI sections

**Works with:**
- `NavigationSplitView` detail columns
- Sidebars and inspectors
- Toolbars and navigation bars
- Custom overlay controls

```swift
Image(imageName)
    .resizable()
    .aspectRatio(contentMode: .fill)
    .frame(minWidth: 0, maxWidth: .infinity, minHeight: 0, maxHeight: .infinity)
    .backgroundExtensionEffect()
```

### Scroll edge effects

Scroll edge effects transition between scrolling content and stationary controls such as toolbars.
SwiftUI chooses an automatic style by platform and context; customize it only when the default does not
fit.

```swift
ScrollView { content }
    .scrollEdgeEffectStyle(.soft, for: .top)
    .scrollEdgeEffectHidden(true, for: .bottom)
```

Styles are `.automatic`, `.hard`, and `.soft`; pass `nil` to restore automatic behavior. UIKit provides
`UIScrollEdgeElementContainerInteraction` for overlay descendants that should shape a scroll edge effect.

For a custom bar rather than a system toolbar, use `safeAreaBar` (iOS 26+, macOS 26+, tvOS 26+, watchOS 26+,
visionOS 26+). It insets the content, adjusts the safe area, and shapes the scroll edge effect around the bar:

```swift
content
    .safeAreaBar(edge: .bottom) {
        PlaybackBar()
    }
```

**Best practices:**
- Test legibility across different images, themes, and contexts
- Combine with gradients for text readability over extended backgrounds
- Use with `.safeAreaInset()` for overlay content positioning
- Pairs well with `.clipShape()` for rounded corners

---

## Button Styles

### .glass

Standard glass button for secondary/common actions:

```swift
Button("Action") { }
    .buttonStyle(.glass)
```

### .glassProminent

Prominent glass button with full tint surface for primary actions:

```swift
Button("Save") { }
    .buttonStyle(.glassProminent)
```

**Note:** `.confirmationAction` toolbar placement auto-applies `.glassProminent`.

### .glass(_:)

Parameterized variant — pass any `Glass`, including a tint or the clear variant:

```swift
Button("Play") { }
    .buttonStyle(.glass(.clear))

Button("Tint") { }
    .buttonStyle(.glass(.regular.tint(.blue)))
```

### Native equivalents

- UIKit: `UIButton.Configuration.glass()`, `.prominentGlass()`, `.clearGlass()`, `.prominentClearGlass()`
- AppKit: `NSButton.BezelStyle.glass`
- tvOS: both SwiftUI glass button styles apply the effect whether or not the button has focus

**macOS note:** Apply `.tint(.clear)` on glass buttons for proper rendering:

```swift
Button("Action") { }
    .buttonStyle(.glass)
    #if os(macOS)
    .tint(.clear)
    #endif
```

---

## Toolbar Glass APIs

Toolbar items automatically receive glass styling when compiled with Xcode 26.

### ToolbarSpacer

```swift
ToolbarSpacer(.flexible)  // Expands to fill available space
ToolbarSpacer(.fixed)     // System-default fixed spacing
```

`ToolbarSpacer` is available on iOS 26, iPadOS 26, Mac Catalyst 26, and macOS 26. UIKit and AppKit use
`UIBarButtonItem.fixedSpace(_:)` and `NSToolbarItem.Identifier.space`.

### ToolbarItemGroup

Groups related actions with consistent glass styling:

```swift
ToolbarItemGroup {
    Button("Draw", systemImage: "pencil") { }
    Button("Erase", systemImage: "eraser") { }
}
```

### Placement-Driven Styling

| Placement | Auto-applied style |
|-----------|-------------------|
| `.confirmationAction` | `.glassProminent` |
| `.cancellationAction` | Standard glass |
| `.primaryAction` | Grouping-capable |
| `.topBarLeading` / `.topBarTrailing` | Navigation areas |
| `.secondaryAction` | Overflow menu (macOS) |

### Removing Title

```swift
.toolbar(removing: .title)
```

### Hiding Items

Hide the `ToolbarItem`, not its contents — hiding only the content leaves an empty glass pill:

```swift
ToolbarItem {
    Button("Favorite", systemImage: "heart") { }
}
.hidden(isEditing)   // ToolbarContent.hidden(_:)

// UIKit: barButtonItem.isHidden = isEditing
// AppKit: toolbarItem.isHidden = isEditing
```

---

## TabView Glass APIs

### Automatic Glass

Tab bars auto-adopt glass when compiled with Xcode 26 SDK. No code changes needed.

### Tab Bar to Sidebar

```swift
TabView { /* tabs */ }
    .tabViewStyle(.sidebarAdaptable)
```

UIKit equivalent: `UITabBarController.Mode.tabSidebar` (iOS/iPadOS 18+, visionOS 2+).

### Tab Bar Minimize

```swift
TabView { /* tabs */ }
    .tabBarMinimizeBehavior(.onScrollDown)
```

### Bottom Accessory

iOS, iPadOS, and Mac Catalyst only. On iPhone it sits above the tab bar at normal size and inline when the
bar is collapsed; read `\.tabViewBottomAccessoryPlacement` to adapt its content.

```swift
.tabViewBottomAccessory {
    Button("Quick Action") { }
        .glassEffect(.regular.interactive())
}
```

### Search Role Tab

```swift
Tab("Search", systemImage: "magnifyingglass", role: .search) {
    SearchView()
}
```

---

## Sheet Glass APIs

Partial-height sheets auto-receive Liquid Glass background.

**Requirements:**
- Specify at least one partial-height detent: `.presentationDetents([.medium, .large])`
- Do NOT use `.presentationBackground()` - system handles glass automatically
- View must be inside `NavigationStack` or `NavigationSplitView` for transitions

### Morphing Sheet Transitions

```swift
@Namespace private var transition

// Source button
Button("Info", systemImage: "info") { showSheet = true }
    .matchedTransitionSource(id: "info", in: transition)

// Sheet content
.sheet(isPresented: $showSheet) {
    InfoView()
        .navigationTransition(.zoom(sourceID: "info", in: transition))
        .presentationDetents([.medium, .large])
}
```

---

## visionOS Glass Background APIs

SwiftUI's control-oriented glass APIs are **unavailable on visionOS**: `glassEffect`, `Glass`,
`GlassEffectContainer`, `glassEffectID`, `glassEffectUnion`, `glassEffectTransition`, and
`.buttonStyle(.glass)` all carry `@available(visionOS, unavailable)`. Custom spatial glass uses
`glassBackgroundEffect`, which fills a view's background and affects z-axis layout. Apply it to an explicit
`ZStack` when the glass should enclose a composed group, rather than an individual child.

```swift
view.glassBackgroundEffect(displayMode: .implicit)
view.glassBackgroundEffect(in: .rect(cornerRadius: 24), displayMode: .always)
view.glassBackgroundEffect(.feathered(padding: 12, softEdgeRadius: nil),
                           in: .rect(cornerRadius: 24), displayMode: .always)
```

`GlassBackgroundDisplayMode` supports `.always`, `.implicit` (only when not already in glass), and
`.never`. `GlassBackgroundEffect` includes `.automatic`, `.plate`, and `.feathered` configurations.

`glassBackgroundEffect` is visionOS 1.0+; the custom-effect overload that takes a `GlassBackgroundEffect`
is visionOS 2.4+. `backgroundExtensionEffect` and `UIBackgroundExtensionView` are also available on
visionOS, unlike the rest of the glass family.

---

## UIKit and AppKit APIs

Use native framework APIs when building UIKit or AppKit interfaces:

- UIKit: `UIGlassEffect(style:)` is a `UIVisualEffect` with `isInteractive` and `tintColor`; styles are
  `UIGlassEffect.Style`. Put it in a `UIVisualEffectView`. iOS/iPadOS/Mac Catalyst/tvOS 26 only — not visionOS.
- UIKit: `UIGlassContainerEffect` combines multiple glass elements; set its `spacing`, then nest
  `UIVisualEffectView`s configured with `UIGlassEffect` in the container's `contentView`.
- UIKit: `UIButton.Configuration.glass()`, `.prominentGlass()`, `.clearGlass()`, `.prominentClearGlass()`.
- UIKit: `UIBackgroundExtensionView` extends content into its bounds;
  `UIScrollEdgeElementContainerInteraction` lets descendants shape scroll edge effects.
- AppKit: `NSGlassEffectView` hosts content in glass (`contentView`, `cornerRadius`, `style`, `tintColor`,
  `effectIsInteractive`). Set its `contentView`; arbitrary subviews aren't guaranteed a specific z-order
  around the effect. `effectIsInteractive` is macOS 27+ and meant for glass backing interactive controls or
  containing them.
- AppKit: `NSGlassEffectContainerView` merges similar descendant effects within `spacing` and can reduce
  rendering passes.
- AppKit: `NSButton.BezelStyle.glass` gives an AppKit button the glass appearance.
- AppKit: `NSBackgroundExtensionView` extends content to fill its bounds.

```swift
let effect = UIGlassEffect(style: .regular)
effect.isInteractive = true
effect.tintColor = .systemBlue
let effectView = UIVisualEffectView(effect: effect)
```

Prefer SwiftUI's `.glassEffect` for SwiftUI interfaces. Bridge only when integrating native hierarchies or
using capabilities SwiftUI doesn't expose directly.

---

## Accessibility APIs

Liquid Glass adapts to accessibility and appearance settings. Validate actual screens with:

- **Reduced Transparency:** Increases frosting for clarity
- **Increased Contrast:** Applies stark colors and borders
- **Reduced Motion:** Tones down animations/elastic effects
- **Preferred Liquid Glass appearance:** User-selected appearance can affect regular and clear variants

### Manual Override (when needed)

```swift
@Environment(\.accessibilityReduceTransparency) var reduceTransparency

var body: some View {
    content
        .glassEffect(reduceTransparency ? .identity : .regular)
}
```

---
