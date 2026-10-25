# Liquid Glass: Common Pitfalls and Solutions

Known issues, workarounds, and best practices when working with Liquid Glass.

## Table of Contents

1. [Visual Issues](#visual-issues)
2. [Animation Issues](#animation-issues)
3. [Performance Issues](#performance-issues)
4. [Interaction Issues](#interaction-issues)
5. [Layout Issues](#layout-issues)
6. [Availability Issues](#availability-issues)
7. [Migration Issues](#migration-issues)

---

## Visual Issues

### Glass elements look inconsistent near each other

**Problem:** Nearby glass elements have different visual appearances, breaking visual harmony.

**Cause:** Without a shared container, each glass element is rendered independently instead of being
merged with its neighbors. A `GlassEffectContainer` merges glass effect views that are within a specified
proximity of each other and batches their render passes, which is what gives a group its unified look.

**Solution:** Wrap all nearby glass elements in a `GlassEffectContainer`:

```swift
// BAD - inconsistent glass
VStack {
    view1.glassEffect()
    view2.glassEffect()
}

// GOOD - shared sampling region
GlassEffectContainer(spacing: 16) {
    VStack {
        view1.glassEffect()
        view2.glassEffect()
    }
}
```

### Glass button rendering on macOS

**Problem:** Glass buttons appear incorrectly tinted on macOS.

**Solution:** Apply `.tint(.clear)` on macOS:

```swift
Button("Action") { }
    .buttonStyle(.glass)
    #if os(macOS)
    .tint(.clear)
    #endif
```

### Clear glass over bright content is unreadable

**Problem:** `.clear` glass over a bright photo or video leaves text and controls hard to read.

**Cause:** Clear glass is highly translucent by design — it prioritizes the media, not legibility.

**Solution:** Add a dark dimming layer at 35% opacity beneath the glass (HIG's figure). Skip it when the
content beneath is already dark, or when AVKit playback controls supply their own dimming. If the component
carries a lot of text (alerts, sidebars, popovers), use `.regular` instead — it blurs and adjusts luminosity.

### Background extension effect appears to do nothing

**Problem:** `backgroundExtensionEffect()` is applied but nothing extends under the sidebar or inspector.

**Cause:** The effect only mirrors into available safe-area space. Padding on the enclosing `ScrollView` or
`VStack`, or an edge that does not reach the containing view, leaves no space to fill.

**Solution:** Remove padding from the containers and align the view's edges with the containing view. Apply
the modifier before overlays so only the image extends.

### Sheet background conflicts

**Problem:** Custom `.presentationBackground()` conflicts with automatic glass styling.

**Solution:** Remove `.presentationBackground()` on iOS 26. The system applies Liquid Glass automatically for partial-height sheets:

```swift
// BAD
.sheet(isPresented: $show) {
    Content()
        .presentationBackground(.ultraThinMaterial) // Remove this
        .presentationDetents([.medium, .large])
}

// GOOD
.sheet(isPresented: $show) {
    Content()
        .presentationDetents([.medium, .large])
    // System handles glass background
}
```

### Text illegibility over glass backgrounds

**Problem:** Text is hard to read when overlaid on `backgroundExtensionEffect` images.

**Solution:** Layer gradients over extended backgrounds:

```swift
Image(imageName)
    .backgroundExtensionEffect()
    .overlay(alignment: .bottom) {
        VStack {
            Text("Title").font(.largeTitle).bold().foregroundColor(.white)
        }
        .padding()
    }
    .safeAreaInset(edge: .bottom) {
        textContent
            .background(LinearGradient(
                colors: [.clear, .black.opacity(0.5)],
                startPoint: .top, endPoint: .bottom
            ))
    }
```

---

## Animation Issues

### rotationEffect causes glass shape morphing

**Problem:** When applying `rotationEffect(_:anchor:)` to views with glass effects, the glass shape morphs unpredictably during animation instead of rotating smoothly.

**Guidance:** This behavior can vary by OS and composition. First validate the effect in the target OS and
consider changing the view hierarchy or animation. If UIKit integration is otherwise appropriate, the
UIKit glass effect requires a style:

```swift
struct RotatableGlassView: UIViewRepresentable {
    let rotation: Angle

    func makeUIView(context: Context) -> UIVisualEffectView {
        let effect = UIGlassEffect(style: .regular)
        let view = UIVisualEffectView(effect: effect)
        return view
    }

    func updateUIView(_ uiView: UIVisualEffectView, context: Context) {
        uiView.transform = CGAffineTransform(rotationAngle: rotation.radians)
    }
}
```

### Menu morphing animation glitches

**Problem:** Applying glass effects directly to Menu labels causes animation artifacts - morphing starts/ends as rectangle, then snaps to circle.

**Solution (varies by OS version):**

iOS 26.0-26.0.1:
```swift
Menu { /* items */ } label: {
    Label("More", systemImage: "ellipsis")
}
.glassEffect(.regular.interactive())
```

iOS 26.1+:
```swift
struct GlassMenuStyle: ButtonStyle {
    func makeBody(configuration: Configuration) -> some View {
        configuration.label
            .padding()
            .glassEffect(.regular)
    }
}

Menu { /* items */ } label: {
    Label("More", systemImage: "ellipsis")
}
.buttonStyle(GlassMenuStyle())
```

### GlassEffectContainer breaks Menu morphing

**Problem:** On iOS 26.1, wrapping Menus in `GlassEffectContainer` causes morphing to break (works on 26.0).

**Solution:** Test across OS versions. Consider placing Menu glass effects outside the container or file feedback with Apple.

### Morphing doesn't animate

**Problem:** Elements with `glassEffectID` don't morph when shown/hidden.

**Checklist:**
1. Elements are in the same `GlassEffectContainer`?
2. Each element has `glassEffectID` with the same `@Namespace`?
3. State change is wrapped in `withAnimation`?
4. Views are conditionally rendered (not just hidden/opacity)?

```swift
// BAD - opacity doesn't trigger morph
view.opacity(isVisible ? 1 : 0)
    .glassEffectID("item", in: namespace)

// GOOD - conditional rendering triggers morph
if isVisible {
    view.glassEffect()
        .glassEffectID("item", in: namespace)
}
```

---

## Performance Issues

### Multiple glass effects cause frame drops

**Problem:** Many individual glass effects create excessive `CABackdropLayer` instances, each requiring 3 offscreen textures.

**Solution:** Group glass elements in `GlassEffectContainer`:

```swift
// BAD - 5 separate backdrop layers (15 textures)
ForEach(items) { item in
    ItemView(item: item).glassEffect()
}

// GOOD - shared backdrop layer
GlassEffectContainer(spacing: 16) {
    ForEach(items) { item in
        ItemView(item: item).glassEffect()
    }
}
```

### ScrollView with many glass elements

**Problem:** Long lists with glass effects cause memory pressure and jank.

**Solution:** Use `LazyVStack`/`LazyHStack` and minimize glass elements in scrollable content. Apply glass to floating overlays, not list items:

```swift
// BAD - glass on every list item
ScrollView {
    LazyVStack {
        ForEach(items) { item in
            ItemRow(item: item).glassEffect()  // Too many!
        }
    }
}

// GOOD - glass on floating controls only
ScrollView {
    LazyVStack {
        ForEach(items) { item in
            ItemRow(item: item)  // No glass on content
        }
    }
}
.overlay(alignment: .bottomTrailing) {
    FloatingButton().glassEffect(.regular.interactive())
}
```

---

## Interaction Issues

### Hit-testing failures on glass buttons

**Problem:** Glass effect buttons only register taps on the content (symbol/text), not the entire glass area.

**Solution:** Use `contentShape()` to define the hit region:

```swift
Button("Action") { }
    .padding()
    .glassEffect(.regular, in: .rect(cornerRadius: 16))
    .contentShape(.rect(cornerRadius: 16))
```

### Toolbar item leaves an empty glass capsule

**Problem:** Hiding a toolbar button's contents leaves a blank glass pill or a gap in the toolbar.

**Cause:** The view inside the item was hidden, but the item is still in the toolbar.

**Solution:** Hide the item itself:

```swift
ToolbarItem {
    Button("Favorite", systemImage: "heart") { }
}
.hidden(isEditing)          // SwiftUI: ToolbarContent.hidden(_:)
// UIKit: barButtonItem.isHidden = isEditing
// AppKit: toolbarItem.isHidden = isEditing
```

### Interactive glass not responding

**Problem:** `.interactive()` glass doesn't scale/bounce on interaction.

**Checklist:**
- `.interactive()` exists on macOS but its documented purpose is glass backing interactive **controls**
  (AppKit: `effectIsInteractive`, `API_AVAILABLE(macos(27.0))`). On a macOS glass surface hosting only
  static text, neither hover nor press produces a visible change. The touch- and motion-driven responses
  are iOS-only in practice, because macOS has neither input.
- The view must be interactive (Button, gesture attached)
- Verify the glass modifier is applied correctly: `.glassEffect(.regular.interactive())`

---

## Layout Issues

### backgroundExtensionEffect clipping

**Problem:** Content with `backgroundExtensionEffect` gets clipped by parent views.

**Solution:** Ensure proper frame and ignore safe area:

```swift
Image(imageName)
    .resizable()
    .aspectRatio(contentMode: .fill)
    .frame(minWidth: 0, maxWidth: .infinity, minHeight: 0, maxHeight: .infinity)
    .clipped()
    .backgroundExtensionEffect()

// On the parent ScrollView:
.ignoresSafeArea(edges: .top)
```

### Concentric corners don't compile

**Problem:** `.glassEffect(.regular, in: .rect(cornerRadius: .containerConcentric))` fails to compile.

**Cause:** `.containerConcentric` belongs to `CornerMaskingConfiguration.CornerStyle`, not `CGFloat`.
`cornerRadius:` only accepts a number.

**Solution:** Use `ConcentricRectangle`, or the shape convenience that takes corner styles:

```swift
view.glassEffect(.regular, in: ConcentricRectangle())
view.glassEffect(.regular, in: .rect(corners: .concentric(minimum: 16)))
```

Corners resolve against the container shape. System views supply one; for a custom container add
`.containerShape(...)`, otherwise the corners have nothing to be concentric with and can resolve to zero.

### Glass container sizing

**Problem:** `GlassEffectContainer` doesn't size correctly.

**Solution:** Apply frame constraints to the inner content, not the container:

```swift
GlassEffectContainer(spacing: 16) {
    VStack {
        ForEach(items) { item in
            ItemView(item: item)
                .glassEffect()
                .glassEffectID(item.id, in: namespace)
        }
    }
    .frame(width: 74)  // Frame on inner content
}
```

### Glass elements crowd or stack

**Problem:** Several glass controls packed together, or one glass surface layered on another, read as a single
muddy shape.

**Cause:** Every glass surface draws its own rim and shadow and samples its own backdrop; overlapping them
compounds those edges. HIG also warns against layering Liquid Glass elements on top of each other.

**Solution:** Keep one glass layer per region, use standard spacing rather than tight or negative padding, and
move secondary actions into a menu instead of adding more glass.

---

## Availability Issues

### `glassEffect` doesn't exist on visionOS

**Problem:** The build fails with an availability error for `glassEffect`, `Glass`, `GlassEffectContainer`,
`glassEffectID`, `glassEffectUnion`, `glassEffectTransition`, or `.buttonStyle(.glass)` when targeting visionOS.

**Cause:** The whole SwiftUI glass family is marked unavailable on visionOS. visionOS has its own spatial glass.

**Solution:** Gate the code and use `glassBackgroundEffect` there:

```swift
#if os(visionOS)
content.glassBackgroundEffect()
#else
content.glassEffect(.regular, in: .rect(cornerRadius: 16))
#endif
```

`backgroundExtensionEffect()` and `UIBackgroundExtensionView` are available on visionOS.

### `#available(iOS 26, *)` fails to build for tvOS or watchOS

**Problem:** A view guarded with `if #available(iOS 26, macOS 26, *)` builds and runs on iOS, then fails to
compile for tvOS, watchOS, or visionOS.

**Cause:** `*` means every platform you did not name is available, so the glass branch is compiled for those
targets. Separately, `#available` can never bypass an unconditional unavailability like visionOS's.

**Solution:** Name every versioned platform in the list, and split visionOS at compile time:

```swift
#if os(visionOS)
content.glassBackgroundEffect()
#else
if #available(iOS 26, macOS 26, tvOS 26, watchOS 26, *) {
    content.glassEffect()
} else {
    content.background(.ultraThinMaterial, in: .rect(cornerRadius: 16))
}
#endif
```

### Glass never appears on older Apple TV

**Problem:** tvOS shows no glass, or custom glass buttons look permanently glassy.

**Cause:** Liquid Glass requires Apple TV 4K (2nd generation) or newer; older devices keep the previous
appearance. Separately, `.buttonStyle(.glass)` applies glass regardless of focus on tvOS.

**Solution:** Test on supported hardware, and drive custom control appearance from focus
(`focusable(_:)`, `@FocusState`, `isFocused`) instead of assuming glass is tied to focus.

---

## Migration Issues

### Ship with the new SDK without adopting the new look

**Problem:** A release has to go out built against the Xcode 26 SDKs before the UI redesign is finished, but
the new appearance changes the whole interface.

**Solution:** Add `UIDesignRequiresCompatibility` = `YES` to the Info pane. The system then runs the app with
the previous SDK's UI while you keep reviewing. It is explicitly temporary: the system ignores the key once
you build with the 27-era SDKs (iOS/iPadOS/Mac Catalyst/macOS/tvOS 27+).

```xml
<key>UIDesignRequiresCompatibility</key>
<true/>
```

### Mixing text and icon toolbar items in one group

**Problem:** A glass toolbar group looks uneven.

**Cause:** Items sharing a glass background are read as one control; mixing text labels and icons breaks that
rhythm and HIG explicitly advises against it.

**Solution:** Use icons for common actions and give each one an accessibility label. Keep text-only and
icon-only items in separate groups, or span them apart with a fixed spacer.

### Custom toolbar backgrounds override glass

**Problem:** Existing `.toolbarBackground()` modifiers prevent glass from appearing.

**Solution:** Remove or conditionally disable custom toolbar backgrounds:

```swift
.toolbar { /* items */ }
// Remove: .toolbarBackground(.visible, for: .navigationBar)
// Remove: .toolbarBackground(.ultraThinMaterial, for: .navigationBar)
```

### Old Material usage on navigation elements

**Problem:** `.background(.ultraThinMaterial)` on navigation containers conflicts with glass.

**Solution:** Remove material backgrounds from navigation elements. Glass handles this automatically. Keep materials only on content backgrounds where appropriate:

```swift
// OK - material on content editing background
#if os(macOS)
static let editingBackgroundStyle = WindowBackgroundShapeStyle.windowBackground
#else
static let editingBackgroundStyle = Material.ultraThickMaterial
#endif
```

### Navigation transitions not working

**Problem:** Sheet morphing transitions or navigation transitions break.

**Solution:** Ensure the view is inside `NavigationStack` or `NavigationSplitView`:

```swift
// BAD - no navigation context
.sheet(isPresented: $show) {
    ContentView()
        .navigationTransition(.zoom(sourceID: "id", in: namespace))
}

// GOOD - wrapped in NavigationStack
.sheet(isPresented: $show) {
    NavigationStack {
        ContentView()
    }
    .navigationTransition(.zoom(sourceID: "id", in: namespace))
}
```

### Symbol effect crashes on older OS

**Problem:** `.symbolEffect(.drawOn)` or other iOS 26 symbol effects crash on older OS versions.

**Solution:** Use `@available` checks:

```swift
if #available(iOS 26, macOS 26, tvOS 26, watchOS 26, *) {
    image.transition(.init(.symbolEffect(.drawOn)))
} else {
    image.transition(.opacity)
}
```
