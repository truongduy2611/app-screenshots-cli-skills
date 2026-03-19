---
name: appshots-maestro-capture
description: >
  Screenshot capture automation using Maestro CLI for mobile apps across all frameworks.
  Declarative YAML flows with automatic waits, built-in takeScreenshot, and cross-platform support.
  Replaces manual bash scripting with stable, maintainable capture workflows.
---

# Maestro Screenshot Capture

Maestro is a **declarative, YAML-driven** mobile UI automation tool with cross-framework support (Flutter, SwiftUI, UIKit, Compose, React Native). Use it to navigate apps, interact with elements, and capture screenshots — **without writing bash scripts or manual sleep timers**.

```
Install → Write YAML Flows → Run → Screenshots
```

## Prerequisites

```bash
# Install Maestro CLI (official installer — NOT brew, which installs a different app)
curl -fsSL "https://get.maestro.mobile.dev" | bash

# Add to PATH (the installer adds this to .zshrc, but for current session)
export PATH="$PATH:$HOME/.maestro/bin"

# Verify installation
maestro --version
```

> ⚠️ **Do NOT use `brew install maestro`** — that installs a code editor cask, not the mobile testing CLI.

> **Supported frameworks:** Flutter (3.19+), SwiftUI, UIKit, Jetpack Compose, Android XML, React Native.

## Core Concepts

### Why Maestro over Bash + AXe?

| Problem with Bash/AXe | Maestro Solution |
|---|---|
| Manual `sleep` timers | **Auto-waits** for elements to appear |
| Coordinate-based taps break across devices | Uses **semantic identifiers** |
| No retry on flaky interactions | **Built-in retry** with configurable tolerance |
| iOS Simulator only | **iOS + Android** with same YAML flow |
| Long imperative scripts | **Short declarative YAML** |

### Element Selection Priority

Maestro finds elements in this order (use the most specific one available):

1. **`id:`** — Matches accessibility identifiers ← **preferred**
   - Flutter: `Semantics(identifier: '...')`
   - SwiftUI: `.accessibilityIdentifier("...")`
   - UIKit: `.accessibilityIdentifier = "..."`
   - Compose: `Modifier.semantics { testTag = "..." }`
   - Android XML: `android:tag="..."`
   - React Native: `testID="..."`
2. **`text:`** — Matches visible text on screen
3. **`label:`** — Matches accessibility label
4. **Relative selectors** — `below:`, `above:`, `leftOf:`, `rightOf:`, `containedIn:`

See the **`appshots-accessibility-ids`** skill for detailed per-framework identifier setup.

## Writing a Capture Flow

Create `.maestro/` directory at your project root. Each flow is a `.yaml` file.

### Basic Flow: `capture_home.yaml`

```yaml
appId: com.example.myapp     # Your app's bundle ID

---
- launchApp

# Home screen
- takeScreenshot: 01_home

# Navigate to a tab
- tapOn:
    id: "tab_search"          # Uses accessibility identifier
- takeScreenshot: 02_search

# Scroll down
- scroll
- takeScreenshot: 03_scrolled
```

### Advanced Flow: `capture_detail.yaml`

```yaml
appId: com.example.myapp

---
- launchApp

# Scroll to find a card
- scrollUntilVisible:
    element:
      id: "card_featured_item"
    direction: DOWN
    timeout: 10000

# Tap the card
- tapOn:
    id: "card_featured_item"

# Wait for detail screen to load
- tapOn:
    id: "play_button"

# Wait for content to appear
- extendedWaitUntil:
    visible: "SCORE"
    timeout: 8000

- takeScreenshot: detail_gameplay
```

### Flow with Conditional Waits

```yaml
appId: com.example.myapp

---
- launchApp

# Navigate to a feature
- tapOn:
    id: "feature_camera"

# Tap a sample item
- tapOn:
    id: "sample_item_0"

# Wait for processing to finish (look for a result element)
- extendedWaitUntil:
    visible:
      id: "result_item_0"
    timeout: 15000

# Tap result to show detail popup
- tapOn:
    id: "result_item_0"

# Wait for popup to animate in
- extendedWaitUntil:
    visible:
      id: "detail_popup"
    timeout: 5000

- takeScreenshot: feature_detail_popup
```

## Running Flows

### Single flow

```bash
maestro test .maestro/capture_home.yaml
```

### All flows in directory

```bash
maestro test .maestro/
```

### Specific device

```bash
maestro test --device <UDID> .maestro/capture_home.yaml
```

### Custom output directory

```bash
maestro test --test-output-dir ./screenshots/raw .maestro/capture_home.yaml
```

## Multi-Locale Screenshot Capture

### Strategy: One Simulator Per Locale (Parallel)

For fastest captures, create a dedicated simulator per locale and run flows in parallel.

```bash
#!/bin/bash
# scripts/maestro-multi-locale.sh
# Usage: bash scripts/maestro-multi-locale.sh [locale1] [locale2] ...
# Example: bash scripts/maestro-multi-locale.sh en vi ja

set -e

DEVICE_TYPE="com.apple.CoreSimulator.SimDeviceType.iPhone-16-Pro-Max"
RUNTIME="com.apple.CoreSimulator.SimRuntime.iOS-18-3"
APP_PATH="build/ios/iphonesimulator/Runner.app"
BUNDLE_ID="com.example.app"
FLOW_DIR=".maestro"
OUTPUT_BASE="./screenshots/raw"

# Configuration
IS_NATIVE_IOS=false  # Set to true if app uses system locale
BUILD_APP=false      # Set to true to build the app before capturing

LOCALES="${@:-en}"

# Map locale codes to Apple format
get_apple_locale() {
  case "$1" in
    en) echo "en_US" ;;
    vi) echo "vi_VN" ;;
    ja) echo "ja_JP" ;;
    ko) echo "ko_KR" ;;
    zh) echo "zh_CN" ;;
    de) echo "de_DE" ;;
    fr) echo "fr_FR" ;;
    es) echo "es_ES" ;;
    pt) echo "pt_BR" ;;
    *) echo "${1}_${1^^}" ;;
  esac
}

capture_locale() {
  local locale="$1"
  local apple_locale
  apple_locale=$(get_apple_locale "$locale")
  local lang="${locale}"
  local sim_name="Screenshot_${locale}"
  local output_dir="${OUTPUT_BASE}/${locale}"

  echo "[$locale] Creating simulator..."
  local udid
  udid=$(xcrun simctl create "$sim_name" "$DEVICE_TYPE" "$RUNTIME" 2>/dev/null || \
         xcrun simctl list devices | grep "$sim_name" | grep -oE '[A-F0-9-]{36}' | head -1)

  echo "[$locale] UDID: $udid"

  # Boot and set locale
  xcrun simctl boot "$udid" 2>/dev/null || true
  sleep 2
  
  if [ "$IS_NATIVE_IOS" = true ]; then
    # Native iOS: change system locale
    xcrun simctl spawn "$udid" defaults write "Apple Global Domain" AppleLocale "$apple_locale"
    xcrun simctl spawn "$udid" defaults write "Apple Global Domain" AppleLanguages -array "$lang"
    xcrun simctl shutdown "$udid"
    sleep 1
    xcrun simctl boot "$udid"
    sleep 5
  else
    # Flutter / Shared Prefs: write directly to app preferences
    local prefs_dir=$(xcrun simctl get_app_container "$udid" "$BUNDLE_ID" data)/Library/Preferences
    defaults write "$prefs_dir/$BUNDLE_ID" selected_locale -string "$lang"
    # No reboot needed for app-level locale
  fi

  # Install app
  xcrun simctl install "$udid" "$APP_PATH"

  # Run Maestro flows
  mkdir -p "$output_dir"
  maestro test \
    --device "$udid" \
    --test-output-dir "$output_dir" \
    "$FLOW_DIR/"

  echo "[$locale] Capture complete → $output_dir"

  # Cleanup: delete the temporary simulator
  xcrun simctl shutdown "$udid" 2>/dev/null || true
  xcrun simctl delete "$udid" 2>/dev/null || true
}

if [ "$BUILD_APP" = true ]; then
  # Build the app first (adjust for your framework)
  # Flutter:        flutter build ios --simulator
  # Xcode (native): xcodebuild -scheme MyApp -sdk iphonesimulator -derivedDataPath build
  # React Native:   npx react-native build-ios --mode=Release --simulator
  echo "Building app for simulator..."
  # ⬇ Replace with your framework's build command
  flutter build ios --simulator
fi

# Run captures (parallel for speed)
for locale in $LOCALES; do
  capture_locale "$locale" &
done
wait

echo "All captures complete! Check $OUTPUT_BASE"
```

## Maestro Studio (Interactive Discovery)

Use `maestro studio` to visually inspect your app's UI tree and generate YAML commands interactively:

```bash
maestro studio
```

This launches a browser-based UI where you can:
- **Click elements** to see their identifiers, text, and labels
- **Generate YAML** by performing actions in the studio
- **Verify selectors** before writing them into flow files

> This is the Maestro equivalent of `axe describe-ui`, but visual and interactive.

## Agent Guidelines

### Thinking Process

When given a task like "capture screenshots of my app across 3 locales":

1. **Check for `.maestro/` directory** — Reuse existing flows if present
2. **Discover identifiers** — Run `maestro studio` or inspect code for accessibility identifiers (see `appshots-accessibility-ids` skill)
3. **Write YAML flows** — One flow per logical screen group (home, detail, settings)
4. **Test single locale first** — `maestro test .maestro/` on the default simulator
5. **Scale to multi-locale** — Use `scripts/maestro-multi-locale.sh en vi ja`
6. **Hand off to design** — Switch to `appshots-design-workflow` skill for framing and styling

### Key Tips

- **Always use `id:` over `text:`** — Text changes across locales, identifiers don't
- **Use `extendedWaitUntil`** — For screens that need loading time (network, animations)
- **Use `scrollUntilVisible`** — Instead of manual swipe coordinates
- **One flow per screen group** — Keeps flows focused and debuggable
- **Use `--test-output-dir`** — To control exactly where screenshots land

### When to Fall Back to AXe

Maestro covers 95% of use cases. Use AXe (`axe describe-ui`, `axe tap`) only when:
- You need the **raw accessibility tree JSON** for debugging
- Maestro doesn't support a specific simulator interaction (e.g., hardware buttons)
- You're working in a CI environment that doesn't support Maestro

## Reference: Maestro Commands for Screenshots

| Command | Purpose |
|---|---|
| `launchApp` | Launch the app (auto-waits for ready) |
| `tapOn:` | Tap an element by id, text, or label |
| `longPressOn:` | Long press an element |
| `scroll` | Scroll down on current screen |
| `scrollUntilVisible:` | Scroll until an element is found |
| `swipe:` | Swipe in a direction (LEFT, RIGHT, UP, DOWN) |
| `inputText:` | Type text into focused field |
| `takeScreenshot:` | Save screenshot with given name |
| `extendedWaitUntil:` | Wait for element to appear (with timeout) |
| `assertVisible:` | Assert an element is on screen |
| `back` | Press back button (Android) |
| `hideKeyboard` | Dismiss the keyboard |
| `clearState` | Clear app data |
| `stopApp` | Force stop the app |
