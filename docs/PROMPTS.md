# TimeBack — Prompt Collection

> The exact prompts used to build TimeBack from scratch with Claude Code.
> These prompts are organized by development phase and can be used sequentially to recreate the app.

---

## How to Use

1. Open Claude Code in your project directory
2. Follow the prompts in order (Phase 1 → Phase 12)
3. Each prompt builds on the previous — don't skip phases
4. Test on a real device after each phase (Screen Time API requires physical device)

> **Important:** You need an Apple Developer account with FamilyControls capability enabled.

---

## Phase 1: Project Setup & Architecture

### Prompt 1.1 — Create Project
```
Create a new iOS SwiftUI project called "TimeBack" — a screen time management app using Apple's FamilyControls/ManagedSettings/DeviceActivity APIs.

Set up:
- Main app target (iOS 18+)
- 4 Foundation extensions: DeviceActivityMonitor, ShieldConfiguration, ShieldAction, DeviceActivityReport (ExtensionKit)
- 1 Widget extension
- App Group: "group.com.yourteam.timeback"
- FamilyControls entitlement on all targets
- DI container pattern with @Observable
- AppGroupStore for cross-process JSON file storage (NOT UserDefaults — it doesn't sync across extensions)
```

### Prompt 1.2 — Data Models
```
Create the core data models:

1. AppRule — daily usage limit with these features:
   - App/category/website selection (FamilyActivitySelection serialization)
   - Daily limit (minutes, with per-weekday customization)
   - Break mode (continuous usage → forced break → auto-resume)
   - On-demand access (configurable "continue for X minutes" on Shield page)
   - Enable/disable state, shield state

2. ScheduleRule — time-based blocking:
   - Start/end time, repeat days (weekday set)
   - Cross-midnight detection
   - Independent app selection

3. ZoneRule — location-based blocking:
   - Coordinates, radius (50-1000m)
   - Reverse geocoded label
   - Active state tracking

4. ShieldConfig — Shield UI customization:
   - Title, message, button text
   - Icon selection (5 options)
   - Unlock method (immediate / delayed with configurable seconds)

5. Weekday enum — shared across models

All models must be Codable with backward-compatible defaults.
```

---

## Phase 2: Rule Engine & DeviceActivity

### Prompt 2.1 — RuleStore
```
Implement RuleStore as an @Observable @MainActor class conforming to RuleStoreProtocol.

It should:
- CRUD operations for rules, schedules, zones
- activateRule() — register DeviceActivity events on a serial background queue
- Events: limit threshold, break checkpoints (max 10), usage tracking (every 30 min up to 12h)
- Custom days: register 7 per-weekday limit events when customDaysEnabled
- deactivateRule() — stop monitoring + remove ManagedSettingsStore shield
- forceShield/forceUnshield — manual control
- toggleRule — enable/disable with full state cleanup
- deleteRule — clean up all orphaned state (shieldedRuleIDs, reShieldTimers, breakEndTimes, distractionCounts, ruleUsage)
- checkReShieldTimers — re-apply shield after on-demand grace period expires
- refreshUsage — read usage data from Report extension or Monitor fallback
```

### Prompt 2.2 — Monitor Extension
```
Implement DeviceActivityMonitorExtension with these event handlers:

eventDidReachThreshold:
- Parse event name suffixes: -limit, -limit-{weekday}, -break-{n}, -usage-{n}
- Per-weekday events: only act if weekday matches today
- Limit event: apply Shield + notification (with dedup guard using isCurrentlyShielded)
- Break event: handleBreak() with duration, auto-resume notification, break end time storage
- Usage event: write checkpoint to ruleUsage in AppGroupStore

intervalDidStart:
- Rules: midnight reset (scoped to specific rule, not global)
- Schedules: time-of-day check before applying Shield (prevent false triggers on re-registration)

intervalDidEnd:
- Schedules: remove Shield + clean activeScheduleIDs
```

---

## Phase 3: Dashboard UI

### Prompt 3.1 — Rule Cards
```
Create the Dashboard with a horizontal card carousel:

RuleCard should display:
- Title + app count
- App icon row (max 5, overflow "+N")
- Activity Ring gauge (300° arc, gap at bottom, white→purple gradient)
- Parameter rows: Daily Limit, Break Mode, On Demand (always shown, greyed when off)
- Shield action button (Block Now / Unblock)
- Enable/disable toggle

Interactions:
- Left/right swipe to navigate cards
- Tap parameter rows → inline edit sheets
- Swipe-up to delete (show hint at 15%, animate fly-out at 40%, then guardian challenge)
- Guardian passcode required for: disable, delete, edit, unblock
```

---

## Phase 4: Shield Extensions

### Prompt 4.1 — Shield UI
```
Implement ShieldConfigurationDataSource:
- Custom dark background with frosted glass blur
- Dynamic icon from ShieldConfig
- Title + message from ShieldConfig
- Primary button: always shown ("OK, Take a Break")
- Secondary button: only shown when onDemandEnabled on the shielded rule
- Button text: "Continue for X min" (X from rule's onDemandMinutes)
- Distraction count: show "Distractions today: N" when N >= 1
- Find the correct rule by checking: AppRules (isCurrentlyShielded && onDemandEnabled) → ScheduleRules → ZoneRules

CRITICAL: NSExtensionPointIdentifier must be "com.apple.ManagedSettingsUI.shield-configuration-service"
```

### Prompt 4.2 — Shield Actions
```
Implement ShieldActionDelegate:
- Primary button → .close (keep blocking)
- Secondary button → unlock flow:
  1. Check if ANY shielded rule has onDemandEnabled — if not, return .close
  2. Increment distraction count
  3. Schedule re-shield timer (don't overwrite earlier timer)
  4. Based on unlock method: immediate → .defer, delayed → wait N seconds then .defer
- Remove only the specific app token from ManagedSettingsStore
- Re-shield timer uses per-rule onDemandMinutes (not hardcoded 15)
```

---

## Phase 5: Schedule & Zone Features

### Prompt 5.1 — Schedules
```
Create AddScheduleView with:
- App selection (FamilyActivityPicker)
- Name field (optional, defaults to "New Schedule")
- Start/end time pickers with cross-midnight detection
- Repeat days: preset buttons (Weekdays/Weekends/Every Day) + individual toggles
- On-demand access configuration
- Preview card showing computed description

ScheduleCard showing: time range, duration, repeat pattern, app icons, status badge
ScheduleListView: same carousel pattern as Dashboard with swipe-delete
```

### Prompt 5.2 — Geofences
```
Create AddZoneView with two-phase interaction:

Phase 1 (Pick Location):
- Full-screen Apple Maps with center crosshair
- Address search with MKLocalSearchCompleter (max 6 suggestions)
- Current location button
- Radius presets: 50m, 100m, 200m (default), 500m, 1000m
- Reverse geocoding (debounced 0.8s)
- "Confirm Location" button

Phase 2 (Configure Rule):
- Map shrinks to 1/3 height
- Name field (auto-filled from geocode)
- App selection
- On-demand access
- "Change Location" to go back

ZoneLocationManager: CLLocationManager wrapper with idempotent enter/exit handling
Clean up map Metal resources on dismiss (onDisappear + isMapVisible flag)
```

---

## Phase 6: Settings & Security

### Prompt 6.1 — Passcode System
```
Implement two-level passcode system:

1. App Passcode (6-digit PIN + Face ID/Touch ID):
   - Locks the app itself
   - Setup: enter → confirm flow
   - Biometric toggle requires biometric verification to enable/disable

2. Guardian Passcode (separate 6-digit PIN):
   - Held by parent/accountability partner
   - 3-step wizard: intro → enter → confirm
   - Required for: disable rules, delete rules, edit rules, unblock

GuardedViewModifier:
- Check guardian hash first → passcode hash → execute directly
- Uses fullScreenCover presentation
- Reset showEntry on success AND cancel (prevent re-present bug)
```

### Prompt 6.2 — Shield Settings
```
Create ShieldSettingsView with:
- Icon picker (5 options: hourglass, moon, leaf, sparkles, shield)
- Text fields: title, message, primary button, secondary button
- Unlock method: immediate or delayed (slider 5-120 sec, step 5)
- iPhone mockup preview (real-time updates)
- Save to AppGroupStore for extensions to read
```

### Prompt 6.3 — Notifications
```
Create NotificationSettingsView with 4 toggles:
- Daily Limit alerts
- Break Mode alerts
- Schedule alerts
- Zone alerts

Each notification send point must check the corresponding toggle from AppGroupStore.
Disabling notifications does NOT affect blocking.
```

---

## Phase 7: Widget

### Prompt 7.1 — Home Screen Widget
```
Create TimeBack widget (small + medium sizes):

Small: ring indicator + blocked count + active rules count
Medium: left stats column + right rule list with status badges

Configuration: max rules shown (1-5), show blocked only toggle
Timeline: 15-minute refresh + immediate refresh on rule changes
Data: read from App Group container, fault-tolerant per-element JSON decoding
```

---

## Phase 8: Dashboard Usage Pipeline

### Prompt 8.1 — Usage Data
```
Implement Dashboard usage data pipeline:

Primary path: DeviceActivityReport extension
- DashboardReportTriggerHost (invisible, always mounted)
- Report extension queries DeviceActivityResults → writes DashboardUsageSnapshot to AppGroupStore
- Exact usage matching iOS Screen Time

Fallback path: Monitor checkpoint data
- When Report extension unavailable (LaunchServices error on dev builds)
- Every 30 min checkpoint → approximate usage
- ±30 min granularity

Snapshot freshness control:
- Keep old data visible while refreshing (no "--" flash)
- Set minimumAcceptedSnapshotDate threshold
- 15 second timeout → fail-open to fallback
- 30-second background poll
```

---

## Phase 9: Localization

### Prompt 9.1 — Multi-language
```
Add full localization support for 7 languages:
English (primary), Simplified Chinese, Traditional Chinese, Japanese, Korean, French, German

1. Create Localizable.xcstrings with all UI strings
2. Create InfoPlist.xcstrings for system permission descriptions
3. All code: use String(localized:) for non-SwiftUI contexts
4. SwiftUI Text("literal") auto-looks up from xcstrings
5. Custom components: use LocalizedStringKey parameter types
6. Ternary expressions: wrap in LocalizedStringKey() or String(localized:)
7. Model defaults: use English values, add zh-Hans as translation
8. Set developmentRegion = en, sourceLanguage = en
9. Add Language entry in Settings → opens iOS system settings
```

---

## Phase 10: App Store Preparation

### Prompt 10.1 — Preflight
```
Prepare for App Store submission:

1. PrivacyInfo.xcprivacy for all 6 targets (UserDefaults CA92.1 + FileTimestamp DDA9.1 + location data)
2. Privacy Policy + Terms of Use pages (host on GitHub Pages)
3. In-app links: privacy policy, terms of use, contact support
4. Restore Purchases button in TipJarView
5. Remove all debug print() statements
6. Remove empty NSAppTransportSecurity from Info.plist
7. Verify all NSUsageDescription keys (FaceID, Location WhenInUse, Location Always)
8. Verify bundle ID hierarchy (extensions are children of main app)
9. Verify deployment target consistency across all targets
```

---

## Bug Fix Prompts (Common Issues)

### Shield Not Loading
```
Shield page shows default iOS "Restricted" instead of custom UI.
Check NSExtensionPointIdentifier in ShieldConfiguration's Info.plist.
Must be "com.apple.ManagedSettingsUI.shield-configuration-service" (NOT "com.apple.ManagedSettings.shield-configuration-data-source").
```

### Rule Toggle Guardian Not Working
```
Guardian passcode not triggering when toggling rule off.
Check: .withGuardedAction() must be on a parent view that contains the environment.
SwiftUI environment doesn't propagate from modifiers applied AFTER the child view that reads it.
Move .withGuardedAction() to MainTabView level.
```

### Duplicate Notifications on Rule Edit
```
Every rule edit triggers limit notification again (if already shielded).
Add dedup guard in Monitor extension: check isCurrentlyShielded before sending notification.
Also check shieldedRuleIDs AppGroupStore key.
In updateRule(): if old rule was already shielded and limit didn't change, skip re-activation.
```

### Map Crash on Zone Dismiss
```
Metal assertion failure when dismissing AddZoneView.
Add onDisappear { isMapVisible = false; geocodeTask?.cancel() }
The MKMapView's Metal drawable gets destroyed while command buffer still references it.
Hiding the map before dismiss allows Metal cleanup.
```
