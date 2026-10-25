# Graceful Degradation and Backward Compatibility

How to ship a Liquid Glass interface that still runs correctly below iOS 26 / macOS 26 / tvOS 26 /
watchOS 26, and on visionOS.

## Two separate axes

Do not conflate these. They fail differently.

| Axis | Question | Effect |
|------|----------|--------|
| **Build SDK** | Which Xcode built the app? | Decides whether system components adopt the new look at all |
| **Runtime OS** | Which OS is the app running on? | Decides whether `#available` branches take the glass path |
| **Deployment target** | Which OS may the app launch on? | Decides whether 26-only symbols compile without guards |

Consequences:

- Built with Xcode 26, running on iOS 18: system components keep the **old** appearance. Custom glass code must
  take its fallback branch, because the symbols do not exist at runtime.
- Built with Xcode 25, running on iOS 26: the app keeps its previous appearance, because linking the older SDK
  opts into compatibility behavior. `UIDesignRequiresCompatibility` is not needed in that case.
- Built with Xcode 26, `UIDesignRequiresCompatibility = YES`, running on iOS 26: old look with new SDK. The key
  is ignored by iOS/iPadOS/Mac Catalyst/macOS/tvOS 27+.

Apps that require 26 can stop here. Everything below is for deployment targets below 26.

## The availability rule that bites

`*` in `#available` means "and every platform not listed is available". It does **not** mean "and everything
else is 26". So this compiles on iOS and fails everywhere else:

```swift
// BAD: the glass branch is considered available on tvOS, watchOS, and visionOS
if #available(iOS 26, macOS 26, *) {
    view.glassEffect()
}
```

On a tvOS or watchOS target the branch is always taken, and `glassEffect` (tvOS/watchOS 26) fails to compile
against an older deployment target. On visionOS it fails unconditionally, because the API is unavailable there
regardless of version.

List every platform that actually gates it:

```swift
if #available(iOS 26, macOS 26, tvOS 26, watchOS 26, *) {
    view.glassEffect()
}
```

iPadOS rides the iOS check; Mac Catalyst follows the iOS version too. `#available` cannot express "unavailable
on a platform" — that is a compile-time question, not a version question.

## visionOS needs `#if`, not `#available`

`Glass` and the `glassEffect` family are `@available(visionOS, unavailable)`. An availability check will never
route around that; you need conditional compilation:

```swift
#if os(visionOS)
content.glassBackgroundEffect(in: .rect(cornerRadius: 16))
#else
if #available(iOS 26, macOS 26, tvOS 26, watchOS 26, *) {
    content.glassEffect(.regular, in: .rect(cornerRadius: 16))
} else {
    content.background(.ultraThinMaterial, in: .rect(cornerRadius: 16))
}
#endif
```

## Put it behind one compat layer

Availability guards inside every view body multiply and drift. Centralize them in a small compatibility file,
then call the helpers everywhere else.

```swift
import SwiftUI

extension View {
    /// Liquid Glass on 26+, material on earlier OSes, spatial glass on visionOS.
    @ViewBuilder
    func glassCompat(cornerRadius: CGFloat = 16) -> some View {
        #if os(visionOS)
        glassBackgroundEffect(in: .rect(cornerRadius: cornerRadius))
        #else
        if #available(iOS 26, macOS 26, tvOS 26, watchOS 26, *) {
            glassEffect(.regular, in: .rect(cornerRadius: cornerRadius))
        } else {
            background(.ultraThinMaterial, in: .rect(cornerRadius: cornerRadius))
                .overlay {
                    RoundedRectangle(cornerRadius: cornerRadius)
                        .strokeBorder(.white.opacity(0.15), lineWidth: 0.5)
                }
        }
        #endif
    }

    @ViewBuilder
    func glassButtonCompat(prominent: Bool = false) -> some View {
        #if os(visionOS)
        buttonStyle(.bordered)
        #else
        if #available(iOS 26, macOS 26, tvOS 26, watchOS 26, *) {
            if prominent { buttonStyle(.glassProminent) } else { buttonStyle(.glass) }
        } else {
            if prominent { buttonStyle(.borderedProminent) } else { buttonStyle(.bordered) }
        }
        #endif
    }
}
```

Two mechanics worth remembering:

- **You cannot wrap modifiers in `if #available`.** Modifiers are applied to a value, so the branch has to
  live in a `@ViewBuilder` helper like the above, not inline in a modifier chain.
- **A helper that returns different style types still compiles** because `@ViewBuilder` erases the branches
  into `_ConditionalContent`. A ternary does not: `.glass` and `.glassProminent` are different types, hence the
  nested `if`.

For larger custom components, annotate the type and gate at the call site instead of branching internally:

```swift
@available(iOS 26, macOS 26, tvOS 26, watchOS 26, *)
@available(visionOS, unavailable)
struct GlassBadge: View { /* uses glassEffect + glassEffectID */ }

// Call site
if #available(iOS 26, macOS 26, tvOS 26, watchOS 26, *) {
    GlassBadge()
} else {
    Badge()
}
```

## Per-feature fallback map

| 26+ API | Keep working below 26 |
|---------|----------------------|
| `.glassEffect` | `.background(.ultraThinMaterial, in: shape)` plus a hairline stroke to imitate the rim |
| `.buttonStyle(.glass)` / `.glassProminent` / `.glass(_:)` | `.bordered` / `.borderedProminent` |
| `GlassEffectContainer`, `glassEffectID`, morphing | Plain layout; `.transition(.opacity)` or `.scale`. Do not encode meaning in the morph |
| `backgroundExtensionEffect` | `.ignoresSafeArea(edges:)` on the image or a gradient scrim behind the bar; keep the image's own clipping |
| `scrollEdgeEffectStyle` / `scrollEdgeEffectHidden` | `.toolbarBackground(.visible, for:)` or a gradient overlay at the edge |
| `safeAreaBar(edge:)` | `.safeAreaInset(edge:)` |
| `ToolbarSpacer` | Standard `ToolbarItem`/`ToolbarItemGroup` placements; `UIBarButtonItem.fixedSpace` in UIKit |
| `tabBarMinimizeBehavior` | Nothing to do — the modifier is inert below iOS 26 |
| `tabViewStyle(.sidebarAdaptable)` (18+) | `.automatic`, or `NavigationSplitView` with its own sidebar |
| `Tab(..., role: .search)` (18+) | `.tabItem` plus `.searchable` |
| `ConcentricRectangle` / `.rect(corners:)` | `ContainerRelativeShape` or an explicit corner radius |
| `matchedTransitionSource` + `.navigationTransition(.zoom)` | Plain `.sheet` presentation — note `.zoom` is also **unavailable on macOS** and needs iOS/tvOS 18+ |
| `.symbolEffect(.drawOn)` | `.opacity` transition — the effect is 26+, same as the rest |
| Icon Composer appearance variants | Older OSes use the default (light) icon; keep it legible on its own |

Fallbacks above assume iOS 15 / macOS 12 or later (the `background(_:in:)` and `safeAreaInset` era). For
earlier deployment targets, use `.background(Material.ultraThinMaterial).clipShape(...)` and `.overlay` instead.

`backgroundExtensionEffect` is the sharpest edge here: below 26 the image no longer bleeds under a sidebar or
toolbar, so any text that relied on the extended scrim can lose contrast. Add the gradient unconditionally if the
layout depends on it.

## UIKit and AppKit

The same rule applies at the framework level — check availability once, return a `UIView`/`NSView` either way.

```swift
func makeGlassBackground() -> UIView {
    if #available(iOS 26, *) {
        let effect = UIGlassEffect(style: .regular)
        effect.isInteractive = true
        return UIVisualEffectView(effect: effect)
    }
    return UIVisualEffectView(effect: UIBlurEffect(style: .systemThinMaterial))
}
```

```swift
func makeMacBackground(content: NSView) -> NSView {
    if #available(macOS 26, *) {
        let glass = NSGlassEffectView()
        glass.contentView = content
        return glass
    }
    let fallback = NSVisualEffectView()
    fallback.material = .hudWindow
    fallback.blendingMode = .withinWindow
    fallback.state = .active
    fallback.addSubview(content)
    return fallback
}
```

- `UIGlassEffect` / `UIGlassContainerEffect` are iOS/iPadOS/Mac Catalyst/tvOS 26 only: they must not be reached
  from a visionOS build.
- `NSGlassEffectView.effectIsInteractive` is macOS **27**+, one release later than the view itself; guard it
  separately.
- `NSButton.BezelStyle.glass` needs its own `#available(macOS 26, *)` check; fall back to `.rounded` or the
  button's contextual bezel style (avoid `.texturedRounded`, which is deprecated).

## Degrade behavior, not just paint

- **Morphing:** glass morphs communicate continuity of place. On older OSes there is no morph, so don't use it
  as the only signal that an item moved or expanded — keep an ordinary presence/absence cue.
- **Tab bar:** minimization and the bottom accessory are iOS 26 only. Never let content depend on the bar being
  collapsed.
- **Scroll edges:** without the system effect, a plain toolbar over scrolling content can lose contrast; keep
  the gradient scrim in the fallback rather than deleting it with the modifier.
- **tvOS focus:** the fallback `.bordered`/`.borderedProminent` styles still react to focus, so prefer them over
  a hand-drawn glass approximation on older Apple TV.
- **Accessibility settings:** the glass branch adapts to Reduce Transparency / Increase Contrast automatically;
  your material fallback does not. Re-check legibility with those settings on for the fallback path too.
- **Motion:** `matchedGeometry` + default animation adds scale/offset; on 26 use `.spring` when you want to opt
  out, and keep the fallback transition short so the two paths feel related.

## Test matrix

| Build | Runtime | Expect |
|-------|---------|--------|
| Xcode 26 | iOS 26 / macOS 26 | Glass everywhere, system components redesigned |
| Xcode 26 | iOS 18 / macOS 15 | No glass; compat fallbacks; old system appearance |
| Xcode 26 + `UIDesignRequiresCompatibility=YES` | iOS 26 | Old appearance; verify it is only a temporary state |
| Xcode 25 | iOS 26 | Old appearance; no glass symbols linked |
| Xcode 26 | visionOS 26 | `glassBackgroundEffect` path compiles; no `glassEffect` |
| Xcode 26 | tvOS 18 / watchOS 11 | Fallback buttons still focus/highlight correctly |

Add a compile-only check to CI for the lowest supported deployment target: availability bugs surface at build
time, not in the simulator.

## Checklist

1. Every 26-only symbol sits behind a complete `#available` list.
2. visionOS handled with `#if os(visionOS)`, never `#available`.
3. One compat layer (`glassCompat`, `glassButtonCompat`, UIKit/AppKit factories), not guards scattered in views.
4. Each 26-only feature has a defined fallback, or is proven not to matter when absent.
5. Fallbacks tested with Reduce Transparency, Increase Contrast, and Reduce Motion.
6. Ship plan: either deployment target 26, or the full fallback set above.
