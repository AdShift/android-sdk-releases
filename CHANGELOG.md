# Changelog

All notable changes to the AdShift Android SDK will be documented in this file.

## [3.1.0] - Unreleased

### Added
- **IAB GPP consent is forwarded** — a GPP string written by your CMP is read and sent with your events once you call `enableGPPDataCollection(true)`. It travels on its own axis, alongside a GDPR decision rather than instead of one, so an app that sets European consent by hand still forwards what its CMP wrote for US users. Every section the CMP wrote is forwarded, national and state alike.

### Changed
- **A CMP axis is read only after you enable it** — `refreshConsent()` used to read and forward a TC string whether or not `enableTCFDataCollection(true)` had ever been called. It now reads only the axes you enabled. If you have never called `enableTCFDataCollection`, a refresh still reads TCF and logs a warning, so upgrading does not quietly stop forwarding your CMP's answer; add the call to keep working when that bridge goes away. An app that passed `false` is left alone.
- **Consent hints are deprecated and decide nothing** — `ConsentHint.TCF` and `ConsentHint.GPP_US_NAT` never selected a source, and naming the GPP hint used to enable GPP for that one call, so a string could appear on one event and be gone from the next. Every hint now behaves as `ConsentHint.AUTO`; enable an axis with `enableTCFDataCollection` or `enableGPPDataCollection`. The parameter stays for source compatibility. `ConsentSource.GPP_US_NAT` keeps its name too — the name is historical and does not narrow what is read.
- **A CMP answer is reported even when it says GDPR does not apply** — a TC string is adopted whenever your CMP publishes a complete set, not only when `IABTCF_gdprApplies` is 1. That value still decides gating; it no longer decides whether the answer is worth reporting. Snapshots and events now carry what the CMP said about users outside GDPR scope.
- **IAB consent is read only where the specs put it** — the SDK reads the preferences file the IAB in-app specs require a CMP to write, plus a few known fallbacks, instead of searching every preferences file the app has. A CMP that writes elsewhere is named with `ConsentOptions.prefsProvider`.
- **Four `ConsentOptions` fields are deprecated no-ops** — `preferTcfWhenBothPresent`, `gppUsNatSid`, `allowPrefsScan` and `enableUsNatMapping`. They kept their place so existing code still compiles; TCF now always owns the European axis when your CMP publishes one, and GPP sections are never filtered by id.

### Fixed
- **A GPP string no longer displaces a TC string your CMP still publishes** — the two axes are independent, and a US signal never takes over the European one.
- **A denial outlives the CMP signal that expressed it** — when a CMP's keys disappear, a consent it produced is forgotten, but a refusal is not: the advertising identifier stays gated instead of being handed back because the keys went away.
- **Reserved values in `IABGPP_GppSID` are no longer read as sections** — `-1` ("no section applies") and `0` ("not yet determined") were taken for section ids.
- **`IABGPP_GppSID` stored as a number is read** — a CMP writing it as an integer rather than a string was ignored, and a store holding any IAB key with an unexpected type no longer fails the whole refresh.

## [3.0.1] - Unreleased

### Fixed
- **Release builds with code shrinking start again** — the SDK now ships the shrinker rules its JSON layer needs, so an app built with `minifyEnabled true` no longer stops at startup with `TypeToken must be created with a type argument`. The rules arrive with the dependency; nothing goes into your own `proguard-rules.pro`. Affects 3.0.0 only.

### Changed
- **`enableTCFDataCollection` documents its real default** — reading IAB TCF consent data is off until you call it with `true`. The behaviour is unchanged; the documentation said the opposite.

## [3.0.0] - 2026-09-10

Major release covering consent handling, device identity and event delivery. Upgrading requires code changes — see the upgrade notes below.

### Changed
- **Consent flags are tri-state** — `AdShiftConsent` exposes `Boolean?` instead of `Boolean`, where `null` means the user has not made a decision. This lets us tell "denied" apart from "never asked" when forwarding consent to partners; under GDPR a `null` flag is treated as no consent.
- **`forGDPRUser` requires all three arguments** — the `hasConsentForAdStorage = false` default and the two-argument Java overload are gone, because that default silently denied ad storage for every integration that omitted it.
- **The SDK ships Auto Backup rules** — a reinstall no longer restores the previous install's identifiers or replays its buffered events, so it is measured as a new install. Apps that declare their own `android:fullBackupContent` or `android:dataExtractionRules` need to merge the SDK exclusions before upgrading.
- **The AdShift device ID is written once** — it is no longer regenerated when ad storage is denied, so `getAdShiftDeviceId()` stays stable for the lifetime of the install. Users are no longer counted more than once after a consent change, and subscription platforms such as RevenueCat and Adapty stitch reliably against it.
- **Consent survives app restarts** — a value passed to `setConsentData` is stored and reapplied on the next launch, together with the advertising identifier gate.
- **Advertising identifiers follow consent for good** — cached GAID and OAID are cleared, the fetch is skipped, and identifiers are stripped from events that were queued before the denial.
- **Limit Ad Tracking is enforced** — when a user opts out at the device level, the advertising ID is skipped and any stored copy is removed.
- **An unreachable backend no longer costs events** — API key validation runs with a valid / invalid / unknown verdict and backoff, and no outcome clears the queue.

### Added
- **Time in app** — every app open reports a lifetime foreground-time counter, which the backend turns into time-in-app and session-length metrics.
- **Meta install referrer and app links** — installs coming from Facebook, Instagram and Lite are attributed through the Meta install referrer, capped at two seconds so it never delays the install event, and `al_applink_data` deep links are parsed for same-session retargeting.
- **Device details** — events carry the device type, hardware model and manufacturer.
- **Delivery reliability** — events are written to disk before any network call and retried from a crash-safe queue with per-endpoint backoff, each carrying an identifier that lets the backend drop duplicates. Server-to-server clicks use the same persistent queue.
- **The opening link is forwarded whole** — `app_install` and `app_open` carry the full deep link as `deeplink_url`, and a server-to-server click carries it as `raw_url`. A link over 2048 bytes is left out rather than shortened, so the server never receives half a link — the event or click is still sent, only without it. This lets attribution be resolved for link formats the SDK does not parse itself, so a campaign no longer has to use a link shape the SDK recognises. The presence of a link is not an attribution claim. Credential-shaped parameters are removed on receipt and are not stored. No integration change is needed.

### Removed
- **`ad_personalization_enabled` at the payload root** — consent travels inside `consent_data` only, which now also carries `ad_storage_enabled`.

### Fixed
- **Legitimate interest counts for TCF purpose 7** — users covered by a legitimate-interest basis under a TCF CMP are no longer treated as having denied measurement.

### Upgrading

Java — pass the third argument:

```java
// before
AdShiftConsent.forGDPRUser(true, false);
// after
AdShiftConsent.forGDPRUser(true, false, false);
```

Kotlin — the flags are nullable now:

```kotlin
// before
val granted: Boolean = consent.hasConsentForAdStorage
// after
val granted: Boolean = consent.hasConsentForAdStorage == true
```

Pass `null` for a flag the user has not decided on.

If your app declares its own backup configuration, the build fails with a manifest merger conflict until you do both of the following — `tools:replace` on its own would re-enable backup of the SDK identifiers:

```xml
<application
    android:fullBackupContent="@xml/my_rules"
    android:dataExtractionRules="@xml/my_data_extraction_rules"
    tools:replace="android:fullBackupContent,android:dataExtractionRules">
```

and exclude `adshift_prefs.xml` and `adshift_event_buffer.json` in your own rule files. The [backup rules guide](https://dev.adshift.com/docs/android-sdk/backup) has the complete example.

---

Releases before 3.0.0 were published without public release notes.
