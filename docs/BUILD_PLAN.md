# TimeBack — Build Plan

> A complete roadmap to build a Screen Time management iOS app from scratch using Claude Code.

## Overview

TimeBack is a full-featured iOS screen time management app built entirely with AI-assisted development (Claude Code). This document contains the complete build plan — every milestone, architectural decision, and feature specification needed to recreate the app.

**Tech Stack:** SwiftUI, FamilyControls, ManagedSettings, DeviceActivity, CoreLocation, WidgetKit, StoreKit 2
**No third-party dependencies.** Pure Apple frameworks only.

---

## Architecture

### Targets (6 + 2 test)

| Target | Type | Purpose |
|--------|------|---------|
| **TimeBack** | iOS App | Main app — rules, schedules, zones, settings |
| **TimeBackDeviceActivityMonitor** | Foundation Extension | Monitors usage, fires limit/break/usage events |
| **TimeBackShieldConfiguration** | Foundation Extension | Renders custom Shield UI (icon, text, buttons) |
| **TimeBackShieldAction** | Foundation Extension | Handles Shield button taps (unlock, re-shield timer) |
| **TimeBackDeviceActivityReport** | ExtensionKit Extension | Generates usage reports for Dashboard |
| **TimeBackWidgetExtension** | Widget Extension | Home screen widget showing rule status |
| **TimeBackTests** | XCTest | Unit tests |
| **TimeBackUITests** | XCTest | UI tests |

### Cross-Process Communication

```
Main App ←→ App Group Container (JSON files) ←→ Extensions
                    ↑
              AppGroupStore.swift
         (FileManager-based, no UserDefaults)
```

All 6 targets share `group.com.Hominexis.timeback.TimeBack` App Group. Data is stored as JSON files (not UserDefaults) because UserDefaults has in-process cache that doesn't sync across extensions.

### Key Patterns

1. **@Observable + @MainActor** — All state containers
2. **Protocol-based DI** — `RuleStoreProtocol`, `ZoneLocationManagerProtocol`
3. **Serial DispatchQueue** — All DeviceActivityCenter XPC calls (1-3 sec blocking)
4. **App Group JSON** — Cross-process data sync
5. **Localized String Catalog** — `.xcstrings` for 7 languages

---

## Milestones

### M0: Project Setup
- Create Xcode project with all 6 extension targets
- Configure App Group entitlement on all targets
- Configure FamilyControls entitlement on all targets
- Set up code signing (development + distribution)
- Create `AppGroupStore.swift` for shared storage
- Create `Dependencies.swift` DI container

### M1: Core Data Models
- `AppRule` — daily limit, break mode, on-demand, custom days
- `ScheduleRule` — time-based blocking with repeat days
- `ZoneRule` — location-based geofencing
- `ShieldConfig` — Shield UI customization
- `Weekday` — shared enum
- `NotificationSettings` — per-category toggles

### M2: Rule Management (CRUD + DeviceActivity)
- `RuleStore` — @Observable state container
- `activateRule()` — register DeviceActivityEvents
- `deactivateRule()` — stop monitoring + remove Shield
- `forceShield()` / `forceUnshield()` — manual control
- `toggleRule()` — enable/disable with Shield cleanup
- `deleteRule()` — full state cleanup (shield, timers, usage, distraction counts)
- `checkReShieldTimers()` — re-apply shield after grace period

### M3: Dashboard UI
- Horizontal card carousel with page dots
- `RuleCard` — ring gauge, parameter rows, action button
- `UsageGaugeView` — 300° arc, Activity Ring style
- Swipe-to-delete gesture (Telegram-style fly-out)
- Inline edit sheets (title, apps, daily limit, break mode, on-demand)
- Guardian passcode gating on all edits

### M4: Monitor Extension
- `eventDidReachThreshold` — handle limit, warning, break, usage events
- Per-weekday limit events (custom days)
- Weekday matching (only fire for today's day)
- Break mode with auto-resume notification
- Midnight reset (scoped per-rule)
- Shield dedup guard (prevent duplicate notifications on rule update)
- Schedule `intervalDidStart` / `intervalDidEnd` with time-of-day check

### M5: Shield Extensions
- `ShieldConfigurationDataSource` — custom icon, title, message, distraction count
- `ShieldActionDelegate` — primary (close) + secondary (unlock) button handling
- Per-rule on-demand minutes
- Re-shield timer (no-overwrite to prevent grace period extension)
- Distraction counter per rule per day

### M6: Schedule Feature
- `AddScheduleView` — time picker, repeat days, overnight detection
- `ScheduleCard` — timeline, duration, status
- `ScheduleListView` — carousel with swipe-delete
- DeviceActivity schedule registration
- `activeScheduleIDs` (separate from rule dedup)

### M7: Geofence Feature
- `AddZoneView` — two-phase: map pin → form
- MapKit with search (MKLocalSearchCompleter)
- Reverse geocoding (debounced 0.8s)
- Radius presets (50m/100m/200m/500m/1000m)
- `ZoneLocationManager` — CLLocationManager wrapper
- Enter/exit region detection with idempotent Shield application
- Zone notification with preference check

### M8: Settings & Security
- `PasscodeSettingsView` — PIN + Face ID/Touch ID
- `GuardianSetupView` — 3-step wizard (intro → enter → confirm)
- `GuardedViewModifier` — environment-based passcode gating
- Priority: Guardian > Passcode > None
- `ShieldSettingsView` — icon picker, text fields, unlock method, iPhone mockup preview
- `NotificationSettingsView` — 4-category toggles
- `TipJarView` — StoreKit 2 consumable IAP

### M9: Widget
- `TimeBackWidget` — small + medium sizes
- `TimeBackWidgetIntent` — configurable (max rules, show blocked only)
- `WidgetAppRule` — lightweight Codable mirror
- 15-minute refresh timeline

### M10: Dashboard Usage Pipeline
- `DashboardReportTriggerHost` — invisible DeviceActivityReport host
- Primary: Report extension writes exact usage via DeviceActivityResults
- Fallback: Monitor checkpoint data (±30 min granularity)
- Snapshot freshness control (no "--" flash on re-enter)
- 30-second background poll

### M11: Localization (7 languages)
- English (primary), Simplified Chinese, Traditional Chinese, Japanese, Korean, French, German
- `Localizable.xcstrings` — 530+ keys
- `InfoPlist.xcstrings` — system permission descriptions
- All code strings via `String(localized:)` or SwiftUI auto-lookup
- Language settings entry in Settings → opens iOS system settings

### M12: App Store Preparation
- `PrivacyInfo.xcprivacy` — all 6 targets
- Privacy Policy + Terms of Use (GitHub Pages)
- In-app legal links + Restore Purchases
- Remove debug prints
- Clean Info.plist (remove empty ATS)
- App Store Connect metadata

---

## Data Flow Diagrams

### Daily Limit Flow
```
User creates rule (30 min limit)
  → RuleStore.addRule()
  → persistRules() (JSON to App Group)
  → activateRule() on monitoringQueue
  → DeviceActivityCenter.startMonitoring(events: [
      -limit at 30min,
      -usage-30, -usage-60, ... -usage-720 (tracking)
    ])
  → ... user uses app ...
  → Monitor Extension: eventDidReachThreshold(-limit)
  → Check isCurrentlyShielded (dedup)
  → applyShield() → ManagedSettingsStore.shield.applications
  → markRuleShielded(true)
  → sendNotification("Daily limit reached")
  → App blocked with custom Shield UI
```

### On-Demand Unlock Flow
```
User taps "Continue for 15 min" on Shield
  → ShieldAction: check onDemandEnabled
  → incrementDistractionCount()
  → scheduleReShield(15 min)
  → unlock() → remove shield tokens → .defer
  → ... 15 minutes pass ...
  → Main App: checkReShieldTimers()
  → forceShield(rule) → re-apply Shield
```

### Guardian Passcode Flow
```
User tries to disable rule (toggle OFF)
  → guardedAction { toggleRule(rule) }
  → GuardedViewModifier checks:
    1. guardianPasscodeHash exists? → show GuardianPasscodeEntryView
    2. passcodeHash exists? → show PasscodeEntryView
    3. Neither? → execute directly
  → On success: execute pending action
  → On cancel: discard
```

---

## File Count Summary

| Category | Files |
|----------|:-----:|
| Views & UI | 23 |
| State Management | 11 |
| Data Models | 8 |
| Extensions | 5 |
| Widget | 3 |
| Theme & Components | 4 |
| Tests | 3 |
| **Total** | **57** |
