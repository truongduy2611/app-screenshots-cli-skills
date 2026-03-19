# App Screenshots CLI Skills

A collection of [Agent Skills](https://github.com/skills) for the **App Screenshots** desktop app CLI (`appshots`). These skills teach AI coding assistants how to create, edit, translate, and export App Store & Google Play screenshots programmatically.

```
npx skills add truongduy2611/app-screenshots-cli-skills
```

## Quick Start

The fastest path to automated App Store & Google Play screenshots:

1. **Prepare your app** → `appshots-accessibility-ids` (add stable accessibility identifiers for your framework)
2. **Capture screenshots** → `appshots-maestro-capture` (write YAML flows, run across locales)
3. **Design & export** → `appshots-design-workflow` (frames, backgrounds, text overlays)
4. **Translate** → `appshots-translation` (AI translation with per-locale overrides)

## Available Skills

### appshots-maestro-capture ⭐ NEW
Screenshot capture automation using **Maestro CLI** with declarative YAML flows.

Use when:
- You need to capture screenshots with **automatic waits** (no manual sleep timers).
- You're working with any mobile framework (Flutter, SwiftUI, UIKit, Compose, React Native).
- You need **cross-platform** capture (iOS Simulator + Android Emulator).
- You want a single YAML flow file instead of bash scripts.

Example:
```
Capture 10 screens of my vocabulary app across en, vi, ja locales using Maestro.
```

---

### appshots-accessibility-ids ⭐ NEW
Adding **stable accessibility identifiers** for screenshot automation across all frameworks.

Use when:
- Your app's interactive elements don't have stable accessibility identifiers yet.
- You need a **naming convention** for identifiers (`screen_element` pattern).
- You want to know **which elements** need identifiers for capture flows.
- You need a **per-framework API reference** (Flutter, SwiftUI, UIKit, Compose, XML Android, React Native).

Example:
```
Add accessibility identifiers to all tappable elements in my home screen for screenshot automation.
```

---

### appshots-automation-pipeline
End-to-end screenshot automation pipeline — tool selection, simulator management, and multi-locale capture.

Use when:
- You need to **choose between Maestro and AXe** for your capture workflow.
- You need to boot simulators and set locales programmatically.
- You want to run parallel multi-locale screenshot captures.

Example:
```
Capture 5 screens of my fitness app across 3 locales (en, ja, ko) using the iPhone 16 Pro Max simulator.
```

---

### appshots-cli-usage

Use when:
- You need the correct `appshots` command or flag combination
- You want JSON output for automation
- You need to check if the app is running

Example:
```
Set the background to a dark color and add a centered white heading.
```

---

### appshots-design-workflow
End-to-end screenshot design workflows: create, style, and export.

Use when:
- You need to build multi-screenshot sets from scratch
- You want to apply backgrounds, frames, text, and overlays
- You need to export screenshots as PNG files

Example:
```
Create 5 App Store screenshots with device frames, centered titles, and dark backgrounds.
```

---

### appshots-translation
AI translation and per-locale customization for multi-language screenshots.

Use when:
- You need to translate text overlays to multiple languages
- You want per-locale font/size/position overrides for CJK or RTL languages
- You need per-locale screenshot images

Example:
```
Translate all screenshot text to Japanese, Korean, and German, then set a Japanese-specific font.
```

---

### appshots-overlay-management
Advanced overlay control: text, image, icon, and magnifier overlays.

Use when:
- You need to add or update text with specific alignment, fonts, and sizing
- You want to layer image or icon overlays on screenshots
- You need to control overlay z-order, copy/paste, and positioning

Example:
```
Add a centered text overlay with Poppins font, then add an app icon below it.
```

---

### appshots-library
Saved design management: save, load, organize into folders, import/export.

Use when:
- You need to save or load screenshot designs
- You want to organize designs into folders
- You need to import/export `.appshots` design files

Example:
```
Save the current multi-screenshot set and export it as an .appshots file.
```

---

### appshots-best-practices
Comprehensive design & copywriting guidelines for screenshots that convert.

Use when:
- You need copy that sells ("Screenshots are ads, not docs")
- You want layout patterns: phone placement, typography ratios, visual rhythm
- You need to verify translations don't overflow across 12+ languages
- You need a pre-export quality checklist

Example:
```
Design 10 App Store screenshots with benefit-focused copy, dark backgrounds, and varied layouts.
```

---

### appshots-background-effects
Background styling: solid colors, gradients, mesh gradients, doodle patterns.

Use when:
- You need gradient or mesh gradient backgrounds
- You want doodle icon patterns as background decoration
- You need transparent backgrounds for compositing

Example:
```
Set a mesh gradient background with blue and purple tones.
```

## Usage

Skills are automatically available once installed. The agent will use them when relevant tasks are detected.

## Skill Structure

Each skill contains:
- `SKILL.md` — Instructions for the agent
- `scripts/` — Helper scripts for automation (optional)
- `examples/` — Sample flows and reference files (optional)
- `references/` — Supporting documentation (optional)

> **Note:** The app supports a maximum of **10 design slots** per multi-screenshot set, matching the App Store's 10-screenshot limit per display size.

## Requirements

- The **App Screenshots** desktop app must be running (for design/export skills)
- The CLI auto-discovers the server port via `~/.config/app-screenshots/server.port`
- Install the CLI: `dart pub global activate --source path packages/app_screenshots_cli`
- **Maestro** (for capture): `curl -fsSL "https://get.maestro.mobile.dev" | bash`
- **AXe** (fallback for capture): `brew install cameroncooke/axe/axe`

## Acknowledgements

This project was inspired by and built upon ideas from the following excellent repositories:
- [rudrankriyam/app-store-connect-cli-skills](https://github.com/rudrankriyam/app-store-connect-cli-skills)
- [ParthJadhav/app-store-screenshots](https://github.com/ParthJadhav/app-store-screenshots)

## License

MIT
