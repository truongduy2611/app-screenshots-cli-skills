---
name: appshots-accessibility-ids
description: >
  Adding stable accessibility identifiers for screenshot automation across all frameworks.
  Covers Flutter, SwiftUI, UIKit, Jetpack Compose, XML Android, and React Native.
  Naming conventions, common widget patterns, and verification strategies.
---

# Accessibility Identifiers for Screenshot Automation

Before any automation tool (Maestro or AXe) can reliably tap elements in your app, those elements need **stable accessibility identifiers**. This skill teaches you how to add them across frameworks.

## Why Accessibility Identifiers?

| Without identifiers | With identifiers |
|---|---|
| Tap by coordinates → breaks on different screens | Tap by `id:` → works everywhere |
| Tap by text → breaks across locales | Identifiers are locale-independent |
| Need trial-and-error | Predictable, documented targets |

## Framework Reference

### Flutter (3.19+)

```dart
Semantics(
  identifier: 'settings_button',       // ← stable, locale-independent
  child: ElevatedButton(
    onPressed: () => openSettings(),
    child: Text(context.l10n.settings), // ← localized, changes per locale
  ),
)
```

> `Semantics.identifier` was contributed by the Maestro team in Flutter 3.19.

### SwiftUI

```swift
Button("Settings") {
    openSettings()
}
.accessibilityIdentifier("settings_button")
```

### UIKit

```swift
button.accessibilityIdentifier = "settings_button"
```

### Jetpack Compose

```kotlin
Button(
    onClick = { openSettings() },
    modifier = Modifier.semantics { testTag = "settings_button" }
) {
    Text(stringResource(R.string.settings))
}
```

> In Compose, use `testTag` inside `Modifier.semantics { }` — Maestro matches this as `id:`.

### Android XML Views

```xml
<Button
    android:id="@+id/settings_button"
    android:contentDescription="@string/settings"
    android:tag="settings_button" />
```

> Maestro uses `android:tag` as the `id:` selector for XML views. `android:id` alone is not enough.

### React Native

```jsx
<TouchableOpacity
  testID="settings_button"
  accessibilityLabel="Settings"
  onPress={openSettings}
>
  <Text>{t('settings')}</Text>
</TouchableOpacity>
```

> React Native uses `testID` which maps directly to Maestro's `id:` selector.

## Naming Conventions

Use a consistent `context_element` pattern across all frameworks:

```
# Navigation
tab_home
tab_search
tab_profile
tab_settings

# Feature cards / buttons
feature_camera
feature_scan
feature_create

# List items (indexed)
list_item_0
list_item_1
search_result_0

# In-screen actions
login_submit_button
onboarding_next_button
dialog_confirm_button
```

### Rules

1. **Always lowercase with underscores** — `settings_button`, not `SettingsButton`
2. **Prefix with screen/context** — Prevents collisions across screens
3. **Use indices for lists** — `list_item_0`, `search_result_0`
4. **Avoid localized text** — Never use translated strings as identifiers
5. **Keep them short but descriptive** — `tab_home`, not `bottom_navigation_bar_home_tab_button`

## Verification

### Using Maestro Studio (All Frameworks)

```bash
maestro studio
```

Opens a browser UI where you can click elements and see their identifiers.

### Using AXe CLI (iOS Only)

```bash
axe describe-ui --udid <SIMULATOR_UDID>
axe describe-ui --udid <SIMULATOR_UDID> | grep "settings_button"
```

### Framework-Specific Debuggers

| Framework | Debug tool |
|---|---|
| Flutter | `showSemanticsDebugger: true` in `MaterialApp` |
| SwiftUI/UIKit | Xcode Accessibility Inspector |
| Android | Layout Inspector → Accessibility pane |
| React Native | Appium Inspector or `accessibilityLabel` logging |

## Minimum Identifiers Checklist

Before running any capture flow, ensure these element types have identifiers:

- [ ] **Navigation tabs** — every tab bar item
- [ ] **Feature cards / CTAs on home screen** — primary actions
- [ ] **Scrollable content items** — cards, list items that need tapping
- [ ] **Key action buttons** — Play, Start, Submit, Confirm
- [ ] **Modal/popup triggers** — elements that open detail views

## Agent Guidelines

### Thinking Process

When asked to "prepare an app for screenshot automation":

1. **Identify the framework** — Flutter, SwiftUI, UIKit, Compose, XML Android, or React Native
2. **Identify screens to capture** — List the screenshots needed
3. **Map navigation paths** — What taps/swipes reach each screen?
4. **Find missing identifiers** — Check which tappable elements lack identifiers
5. **Add identifiers** — Use the correct API for the framework (see table above)
6. **Rebuild and verify** — Build the app, then run `maestro studio` or `axe describe-ui`

### Common Mistakes to Avoid

- ❌ Adding identifiers to non-interactive widgets (labels, dividers)
- ❌ Using localized strings as identifiers
- ❌ Forgetting to rebuild after adding identifiers
- ❌ Using the same identifier on multiple elements on the same screen
- ❌ Using `android:id` without `android:tag` for XML Android views
