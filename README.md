# Liquid Glass - Claude Code Skill

A [Claude Code](https://claude.com/claude-code) skill for building and migrating Apple platform apps with SwiftUI's **Liquid Glass** design system (iOS 26+, macOS 26+, iPadOS 26+).

## What This Skill Does

When installed, Claude Code gains deep knowledge of Apple's Liquid Glass design system introduced at WWDC 2025. It can:

- **Migrate existing apps** to Liquid Glass with a structured, phased workflow
- **Build new apps from scratch** using glass-native patterns
- **Apply all glass APIs** correctly: `.glassEffect()`, `.backgroundExtensionEffect()`, `.buttonStyle(.glass)`, `GlassEffectContainer`, morphing transitions, and more
- **Handle platform differences** across iOS, iPadOS, macOS, watchOS, tvOS, and visionOS
- **Avoid common pitfalls** with tested solutions for known issues

The skill auto-activates when you discuss SwiftUI glass design, or invoke it manually with `/liquid-glass:liquid-glass`.

## Install

### Step 1: Add the marketplace

```
/plugin marketplace add romanbsd/liquid-glass-skill
```

### Step 2: Install the plugin

```
/plugin install liquid-glass@liquid-glass-skills
```

### Step 3: Use it

Ask Claude Code to work with Liquid Glass — it will automatically apply the skill. Or invoke directly:

```
/liquid-glass:liquid-glass
```

## What's Included

| File | Description |
|------|-------------|
| `SKILL.md` | Core instructions — key APIs, quick start, migration summary |
| `references/api-reference.md` | Complete API reference for all glass modifiers and types |
| `references/migration-guide.md` | Phased migration workflow for existing apps, opt-out key, app icon adoption |
| `references/graceful-degradation.md` | Backward compatibility below 26: availability rules, fallback map, test matrix |
| `references/platform-specifics.md` | iOS, macOS, iPadOS, watchOS, tvOS, visionOS details |
| `references/pitfalls-and-solutions.md` | Known issues with tested solutions |
| `examples/landmarks-patterns.md` | Real code patterns from Apple's Landmarks sample app |

## APIs Covered

- `glassEffect(_:in:)` — glass material on any view (not available on visionOS)
- `Glass` struct — `.regular`, `.clear`, `.identity`, `.tint()`, `.interactive()`
- `GlassEffectContainer` — grouped glass with shared sampling
- `glassEffectID(_:in:)` — morphing transitions
- `glassEffectUnion` — combined glass surfaces
- `backgroundExtensionEffect()` — content behind navigation layer
- `.buttonStyle(.glass)` / `.glassProminent` / `.glass(_:)` — glass buttons
- `ToolbarSpacer`, `ToolbarItemGroup` — glass toolbar layout
- `.tabBarMinimizeBehavior(.onScrollDown)`, `.tabViewStyle(.sidebarAdaptable)` — glass tab bars
- `safeAreaBar(edge:alignment:spacing:content:)` — custom bars with scroll edge effects
- `ConcentricRectangle` / `.rect(corners:)` — corners concentric with their container
- Glass sheets with `.presentationDetents`
- `UIDesignRequiresCompatibility` — temporary compatibility mode while migrating
- App icon layers and Icon Composer appearances
- UIKit (`UIGlassEffect`, `UIGlassContainerEffect`, `UIButton.Configuration`) and AppKit (`NSGlassEffectView`, `NSButton.BezelStyle.glass`) equivalents

## Sources

Built from analysis of:
- [Apple: Landmarks - Building an app with Liquid Glass](https://developer.apple.com/documentation/SwiftUI/Landmarks-Building-an-app-with-Liquid-Glass)
- [Apple: Adopting Liquid Glass](https://developer.apple.com/documentation/technologyoverviews/adopting-liquid-glass)
- [Apple: Applying Liquid Glass to custom views](https://developer.apple.com/documentation/SwiftUI/Applying-Liquid-Glass-to-custom-views)
- [Apple HIG: Materials](https://developer.apple.com/design/human-interface-guidelines/materials)
- [WWDC25 Session 323](https://developer.apple.com/videos/play/wwdc2025/323/)
- SwiftUI / UIKit / AppKit SDK interface and headers, cross-checked against platform availability
- Community references and real-world adoption experiences

## License

MIT
