---
name: appshots-automation-pipeline
description: >
  End-to-end screenshot automation pipeline for mobile apps.
  Covers tool selection (Maestro vs AXe), simulator management, app navigation,
  raw screenshot capture across multiple locales, and design hand-off to appshots.
---

# App Screenshot Automation Pipeline

This skill teaches an AI agent how to automate the full App Store screenshot capture workflow:

```
Prepare App (Semantics) → Boot Simulators → Capture (Maestro/AXe) → Design Hand-off
```

## Tool Selection

Choose the right capture tool based on your situation:

```
Need auto-waits, retry logic, or cross-platform capture?
├── YES → Use Maestro (see appshots-maestro-capture skill)
│         ├── Declarative YAML flows
│         ├── Auto-waits, no sleep timers
│         ├── Built-in takeScreenshot
│         ├── Cross-platform (iOS + Android)
│         └── Works with: Flutter, SwiftUI, UIKit, Compose, React Native
│
└── NO → Need raw accessibility tree JSON or iOS-only advanced control?
    ├── YES → Use AXe + bash scripts (see below)
    └── NO → Use Maestro
```

> **Recommended for most apps:** Always start with Maestro. Fall back to AXe only for edge cases.

### When to Use Each Tool

| Scenario | Tool |
|---|---|
| Any mobile app — cross-platform capture | **Maestro** |
| Need auto-waits and retry logic | **Maestro** |
| Need raw accessibility tree JSON | **AXe** (`axe describe-ui`) |
| iOS-only native app (Swift/ObjC) | Either (Maestro or AXe) |
| CI environment without Maestro support | **AXe** |
| Hardware button simulation | **AXe** |

## Prerequisites

```bash
# Option A: Maestro (Recommended — use official installer, NOT brew)
curl -fsSL "https://get.maestro.mobile.dev" | bash
export PATH="$PATH:$HOME/.maestro/bin"

# Option B: AXe (Fallback)
brew install cameroncooke/axe/axe

# Always needed
# xcrun simctl — Built-in with Xcode (no install needed)
```

## Workflow Overview

### Phase 0: Prepare the App

Before capture, ensure your app has stable accessibility identifiers. See the **`appshots-accessibility-ids`** skill for:
- Adding accessibility identifiers to tappable elements (per-framework API reference)
- Naming conventions (`screen_element` pattern)
- Verification with `maestro studio` or `axe describe-ui`

### Phase 0.5: Demo Data & Pre-Capture Setup

Screenshots need the app to look **populated and polished** — empty states, blank lists, and "getting started" screens ruin App Store screenshots.

**Before writing any capture flows, ask the user:**

1. **Does the app need demo/seed data?** — Pre-populated lists, sample content, user profiles, etc.
2. **Does the app have a demo mode or test account?** — Some apps have built-in mechanisms.
3. **Does any content need to be locale-specific?** — Sample names, dates, currencies.
4. **Are there any onboarding/tutorial screens to skip?** — First-launch modals, permission dialogs.
5. **Should the app show a specific state?** — Logged-in, premium unlocked, specific tab selected.

#### Common Setup Strategies

| Strategy | When to use | How |
|---|---|---|
| **Seed database** | App has local DB (SQLite, Hive, Realm) | Pre-populate with sample records via script or test fixture |
| **SharedPreferences / UserDefaults** | Skip onboarding, set flags | Write keys before launch: `xcrun simctl spawn <UDID> defaults write <bundle_id> has_seen_onboarding -bool true` |
| **Test account** | App requires login | Use a dedicated screenshot account with curated content |
| **Demo mode flag** | App supports it | Launch with flag: `xcrun simctl launch <UDID> <bundle_id> --demo-mode` |
| **Mock API** | App fetches from server | Use a local mock server or cached responses |
| **Clear and rebuild** | Ensure clean state | `xcrun simctl uninstall <UDID> <bundle_id>` then reinstall |

#### Example: SharedPreferences Setup (Flutter)

```bash
# Skip onboarding and set demo state before launching the app
UDID="booted"
BUNDLE_ID="com.example.myapp"

# Write to the app's shared preferences via simctl
PREFS_DIR=$(xcrun simctl get_app_container "$UDID" "$BUNDLE_ID" data)/Library/Preferences
defaults write "$PREFS_DIR/$BUNDLE_ID" has_completed_onboarding -bool true
defaults write "$PREFS_DIR/$BUNDLE_ID" demo_data_loaded -bool true
```

> **Key principle:** The agent should always ask about demo data needs BEFORE writing capture flows. Capturing an empty app is wasted effort.

### Phase 0.75: Detect Locale Handling Strategy

Apps handle locales in two fundamentally different ways. **You MUST check the source code** to determine which strategy the app uses — using the wrong one means screenshots won't change language.

#### Strategy A: System-Level Locale (Default)

The app follows the device/simulator language setting.

**How to detect per framework:**

| Framework | System locale indicators |
|---|---|
| **Flutter** | Standard `AppLocalizations` with `MaterialApp.localizationsDelegates`, no custom locale provider |
| **SwiftUI/UIKit** | Uses `NSLocalizedString` / `String(localized:)`, `.lproj` bundles, no in-app picker |
| **Android** | Uses `strings.xml` in `values-ja/`, `values-ko/`, etc., no custom locale logic |
| **React Native** | Uses `i18next` or `react-intl` tied to `I18nManager` / system locale |

**How to switch locale (iOS Simulator):**
```bash
xcrun simctl spawn "$UDID" defaults write "Apple Global Domain" AppleLanguages -array "ja"
xcrun simctl spawn "$UDID" defaults write "Apple Global Domain" AppleLocale "ja_JP"
xcrun simctl shutdown "$UDID" && xcrun simctl boot "$UDID"
```

**How to switch locale (Android Emulator):**
```bash
adb -s emulator-5554 shell "su 0 setprop persist.sys.locale ja-JP; setprop ctl.restart zygote"
```

#### Strategy B: App-Level Locale (Custom)

The app manages its own language setting internally. Changing the simulator/emulator locale has **NO EFFECT**.

**How to detect per framework:**

| Framework | App-level locale indicators |
|---|---|
| **Flutter** | `SharedPreferences` key for locale, custom `LocaleProvider` / `LanguageCubit`, `easy_localization` package, `MaterialApp.locale` set from state |
| **SwiftUI/UIKit** | `UserDefaults` key for language, in-app language picker, custom `LanguageManager` class |
| **Android** | `SharedPreferences` key for locale, custom `LocaleHelper`, in-app language dropdown, `AppCompatDelegate.setApplicationLocales()` |
| **React Native** | `AsyncStorage` key for language, `i18next.changeLanguage()` called from settings screen |

**How to switch locale:**
```bash
# iOS: Write the preference directly before launch
PREFS_DIR=$(xcrun simctl get_app_container "$UDID" "$BUNDLE_ID" data)/Library/Preferences
defaults write "$PREFS_DIR/$BUNDLE_ID" selected_locale -string "ja"

# Android: Write SharedPreferences directly
adb shell "run-as $PACKAGE_NAME cat /data/data/$PACKAGE_NAME/shared_prefs/*.xml"
# Then modify and push back, or use Maestro to tap through the UI
```

```yaml
# Or use Maestro to tap the in-app language picker
- tapOn:
    id: "tab_settings"
- tapOn:
    id: "language_picker"
- tapOn:
    text: "日本語"
```

#### Quick Detection Commands

```bash
# Flutter
grep -r "SharedPreferences.*locale\|SharedPreferences.*language\|easy_localization\|LocaleProvider\|LanguageCubit" lib/

# iOS Native (Swift)
grep -r "UserDefaults.*language\|LanguageManager\|setLanguage" --include="*.swift" .

# Android (Kotlin/Java)
grep -r "SharedPreferences.*locale\|LocaleHelper\|setApplicationLocales" --include="*.kt" --include="*.java" .

# React Native
grep -r "AsyncStorage.*language\|i18next.changeLanguage\|setLanguage" --include="*.js" --include="*.ts" --include="*.tsx" src/
```

> **If unsure, ask the user:** _"Does your app follow the system locale, or does it have its own language picker in settings?"_

### Phase 1: Capture Raw Screenshots

**Option A: Maestro (Recommended)**

See the **`appshots-maestro-capture`** skill for:
- Writing YAML flows with `takeScreenshot`
- Multi-locale parallel capture
- `scrollUntilVisible`, `extendedWaitUntil`, and other smart commands

**Option B: AXe + asc CLI (Fallback)**

Create `.asc/screenshots.json` to define the steps AXe should take in the app.

```json
{
  "version": 1,
  "app": {
    "bundle_id": "com.example.app",
    "udid": "booted",
    "output_dir": "./screenshots/raw"
  },
  "steps": [
    { "action": "launch" },
    { "action": "wait", "duration_ms": 2000 },
    { "action": "screenshot", "name": "01_home" },
    { "action": "tap", "id": "settings_tab" },
    { "action": "wait", "duration_ms": 1000 },
    { "action": "screenshot", "name": "02_settings" }
  ]
}
```

*Note: Use `axe describe-ui` to find the correct `id` and `label` values for the steps.*

#### Parallel Multi-Locale Capture (AXe)

Instead of sequentially rebooting one simulator, map each target locale to a **dedicated simulator UDID** created via `xcrun simctl create`. This allows parallel capture.

```bash
#!/bin/bash
# parallel-capture.sh — Fast multi-locale capture

declare -A LOCALE_UDID=(
  ["en-US"]="UDID_EN_US"
  ["ja-JP"]="UDID_JA_JP"
  ["ko-KR"]="UDID_KO_KR"
)

# Configuration
IS_NATIVE_IOS=false  # Set to true if app uses system locale
BUILD_APP=false      # Set to true to build the app before capturing
BUNDLE_ID="com.example.app"

if [ "$BUILD_APP" = true ]; then
  echo "Building app for simulator..."
  # ⬇ Replace with your framework's build command
  flutter build ios --simulator
fi

# 1. Boot all simulators and set their locales
for LOCALE in "${!LOCALE_UDID[@]}"; do
  (
    UDID="${LOCALE_UDID[$LOCALE]}"
    LANG="${LOCALE%%-*}"            # e.g., 'ja'
    APPLE_LOCALE="${LOCALE/-/_}"    # e.g., 'ja_JP'

    xcrun simctl boot "$UDID" || true
    
    if [ "$IS_NATIVE_IOS" = true ]; then
      # Native iOS: change system locale
      xcrun simctl spawn "$UDID" defaults write NSGlobalDomain AppleLanguages -array "$LANG"
      xcrun simctl spawn "$UDID" defaults write NSGlobalDomain AppleLocale -string "$APPLE_LOCALE"
      
      # Needs a reboot to apply locale changes cleanly
      xcrun simctl shutdown "$UDID"
      xcrun simctl boot "$UDID"
    else
      # Flutter / Shared Prefs: write directly to app preferences
      PREFS_DIR=$(xcrun simctl get_app_container "$UDID" "$BUNDLE_ID" data)/Library/Preferences
      defaults write "$PREFS_DIR/$BUNDLE_ID" selected_locale -string "$LANG"
    fi

    # Install the built app (adjust path for your framework)
    xcrun simctl install "$UDID" build/ios/iphonesimulator/Runner.app
  ) &
done
wait

# 2. Run captures in parallel using the asc plan
for LOCALE in "${!LOCALE_UDID[@]}"; do
  (
    UDID="${LOCALE_UDID[$LOCALE]}"
    asc screenshots run \
      --plan ".asc/screenshots.json" \
      --udid "$UDID" \
      --output-dir "./screenshots/raw/$LOCALE" \
      --output json
      
    echo "Captured $LOCALE on $UDID"
  ) &
done
wait
echo "All raw captures complete."
```

### Phase 2: Design Hand-off

Once the raw screenshots are captured, immediately transition to the `appshots-design-workflow` skill to load these images into the App Screenshots desktop app for framing, backgrounds, text overlays, and PDF/PNG export.

---

## Agent Guidelines

### Thinking Process for AI Agents

When given a scenario like "capture screenshots of my fitness app", follow this process:

1. **Ask about demo data** — Does the app need seed data, a test account, or flags to skip onboarding?
2. **Prepare the app** — Add accessibility identifiers to key elements (see `appshots-accessibility-ids` skill)
3. **Set up demo state** — Seed data, write SharedPreferences flags, set up test account
4. **Choose the capture tool** — Maestro for most apps, AXe for edge cases
5. **Discover the app** — Find the bundle ID, build the app if needed
6. **Boot a simulator** — Pick the right device type for the target display
7. **Write capture flows** — Maestro YAML or asc JSON plan
8. **Run single-locale first** — Verify captures are correct
9. **Scale to multi-locale** — Parallel capture across all target locales
10. **Design** — Switch to `appshots` CLI skills to set up frames, text, backgrounds
11. **Translate** — Switch to `appshots-translation` skill for AI text translation
12. **Export** — Export final screenshots for all locales

### Pre-Capture Questions Checklist

Always ask the user these questions before starting capture:

- [ ] Which screens do you want to capture? (list all screenshots)
- [ ] Does the app need demo/seed data to look populated?
- [ ] Is there a test account or demo mode?
- [ ] Any onboarding or permission dialogs to skip?
- [ ] Which locales do you need? (e.g., en, ja, ko, de)
- [ ] Any specific app state needed? (premium, logged in, specific tab)

### Device Selection Strategy (App Store Connect)

When choosing which simulators to boot and generate screenshots for, follow this device tiering:
- **Required Default (6.9"):** `iPhone 16 Pro Max` or `iPhone 17 Pro Max`. This covers the primary 6.9" App Store Connect requirement and should ALWAYS be captured.
- **Optional Sizes:**
  - **6.5" (Optional):** `iPhone 15 Pro Max` or `iPhone 11 Pro Max`.
  - **5.5" (Optional):** `iPhone 8 Plus` (only if legacy compatibility is explicitly requested).

### Key Tips

- **Prepare identifiers first** — The biggest time sink is missing accessibility identifiers
- **Use Maestro for capture** — Auto-waits eliminate fragile `sleep` timers
- **Use `--id` over coordinates** — Identifiers are stable across locales and screen sizes
- **Test single locale first** — Catch issues before scaling to all locales
- **Status Bar Overrides are Optional** — Setting the simulator time to 9:41 and overriding network/battery states (`xcrun simctl status_bar`) is optional. By default, simulators usually look clean enough, and `appshots` designs often use backgrounds that can mask standard status bars.
