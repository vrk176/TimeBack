# TimeBack Privacy Policy

*Last updated: March 30, 2026*

## Overview

TimeBack ("the App") is developed by an independent developer. We take your privacy seriously. This policy explains how the App handles your data.

**The core principle: TimeBack does not collect, transmit, or store any personal data on external servers. All data remains on your device.**

## Data We Do NOT Collect

- We do **not** collect personal information (name, email, phone number)
- We do **not** collect usage analytics or behavioral data
- We do **not** use advertising SDKs or tracking frameworks
- We do **not** share any data with third parties
- We do **not** use cookies or cross-app tracking
- We do **not** require account creation or login

## Data Stored Locally on Your Device

The following data is stored **exclusively on your device** using Apple's App Group container and is never transmitted:

| Data | Purpose |
|------|---------|
| App usage rules | Your configured time limits, break modes, and on-demand settings |
| Schedule rules | Your configured blocking schedules |
| Geofence rules | Location coordinates and radius for zone-based blocking |
| Shield configuration | Your customized blocking screen appearance |
| Passcode hashes | SHA-256 hashes of your PIN and guardian PIN (original PINs are never stored) |
| Notification preferences | Your per-category notification toggle states |
| Usage checkpoints | Approximate app usage minutes for dashboard display |
| Distraction counts | Number of times "Continue Using" was tapped per day |

## Apple Frameworks & APIs

TimeBack uses the following Apple system frameworks:

### Screen Time API (FamilyControls / ManagedSettings / DeviceActivity)
- Used to monitor app usage time and enforce blocking
- All usage data is processed locally by Apple's system extensions
- TimeBack does **not** have access to your browsing history, message content, or app-specific data
- TimeBack can only see opaque app tokens and aggregate usage durations

### Location Services (CoreLocation)
- Used **only** for the Geofence feature
- Location data is processed locally to determine if you are inside a configured zone
- Location coordinates are stored only in your zone rule configurations on-device
- No location data is transmitted to any server
- You can disable location access at any time in system Settings

### Biometric Authentication (LocalAuthentication)
- Used optionally for Face ID / Touch ID app lock
- Biometric data is handled entirely by Apple's Secure Enclave
- TimeBack never accesses or stores biometric data

### StoreKit (In-App Purchase)
- Used for the optional "Buy Developer a Coffee" tip
- Purchase transactions are handled by Apple
- We do not receive any personal payment information

## Data Retention

- All data is stored on your device only
- Uninstalling the App permanently deletes all data
- There is no cloud backup of TimeBack-specific data
- Daily counters (usage, distraction count) reset automatically at midnight

## Children's Privacy

TimeBack can be used as a parental control tool via the Guardian Passcode feature. The App does not knowingly collect personal information from children. All data remains local to the device.

## Third-Party Services

TimeBack does **not** integrate any third-party analytics, advertising, or tracking services. The only external communication is with Apple's servers for:
- In-App Purchase transaction verification (StoreKit)
- Map tile loading (MapKit, for the Geofence feature)

## Your Rights

Since we do not collect any personal data, there is no personal data to access, modify, or delete from our servers. All data on your device is under your full control and can be deleted by uninstalling the App.

## Changes to This Policy

We may update this Privacy Policy from time to time. Changes will be posted on this page with an updated "Last updated" date. Continued use of the App after changes constitutes acceptance of the updated policy.

## Contact

If you have any questions about this Privacy Policy, please contact us at:

**Email:** [connect@hominexis.com](mailto:connect@hominexis.com)

---

*TimeBack is developed and maintained by an independent developer.*
