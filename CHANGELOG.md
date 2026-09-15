# Changelog

All notable changes to the AdShift Android SDK will be documented in this file.

## [3.1.0] - 2026-09-15

### Added
- **IAB GPP consent is forwarded** — a GPP string written by your CMP is read and sent with your events once you call `enableGPPDataCollection(true)`. It travels on its own axis, alongside a GDPR decision rather than instead of one, so an app that sets European consent by hand still forwards what its CMP wrote for US users. Every section the CMP wrote is forwarded, national and state alike.
- **Third-party sharing is a separate answer** — `setThirdPartySharing(AdShiftThirdPartySharing.optedOut())` records that the user asked you not to share their data onward, and `clearThirdPartySharing()` withdraws the declaration. Mind the polarity, which is the reverse of a consent flag: saying nothing means sharing is allowed, so `allowed()` is a statement you made, not a default to set at startup. The declaration is stored and reapplied on the next launch.
- **A snapshot tells "no CMP" apart from "a CMP nobody answered"** — `ConsentSnapshot.cmpDetected` reports whether a CMP is installed at all, independently of whether it has published a usable answer and of which axes you enabled. `cmpDetected == true` with no string means the user has not answered yet and will; `false` means no CMP is writing, which is worth checking against your integration.
- **`consentRequired` and `consentNotRequired`** — the same two factories as `forGDPRUser` and `forNonGDPRUser`, under names that say what they decide. The old names keep working.

### Changed
- **Non-GDPR users no longer report granted consent** — `forNonGDPRUser()` and `consentNotRequired()` state the scope and nothing else; the three consent flags come back as `null` and go on the wire unset. Nothing changes about what is gated: outside GDPR scope nothing was ever gated on those flags, and every predicate that reads them is guarded by `gdpr_applies`. The old values were a record claiming a consent nobody had collected, which is why they are gone. If you do collect consent outside GDPR scope and want it on record, pass the flags to `AdShiftConsent` directly.
- **GPP presence is reported only after you enable it** — 3.0.0 looked for a CMP's GPP keys on every refresh with no opt-in, so `ConsentSnapshot.gppPresent` could come back `true` and `source` could be `GPP_US_NAT` without you asking for either. Both now require `enableGPPDataCollection(true)`. Unlike the TCF axis below, this one has no bridge: if you read `gppPresent` and do not enable the axis, you get `false`. Nothing stops being forwarded, because 3.0.0 never forwarded a GPP string in the first place — only its presence was visible.
- **A CMP axis is read only after you enable it** — `refreshConsent()` used to read and forward a TC string whether or not `enableTCFDataCollection(true)` had ever been called. It now reads only the axes you enabled. If you have never called `enableTCFDataCollection`, a refresh still reads TCF and logs a warning, so upgrading does not quietly stop forwarding your CMP's answer; add the call to keep working when that bridge goes away. An app that passed `false` is left alone.
- **Consent hints are deprecated and decide nothing** — `ConsentHint.TCF` and `ConsentHint.GPP_US_NAT` never selected a source, and naming the GPP hint used to enable GPP for that one call, so a string could appear on one event and be gone from the next. Every hint now behaves as `ConsentHint.AUTO`; enable an axis with `enableTCFDataCollection` or `enableGPPDataCollection`. The parameter stays for source compatibility. `ConsentSource.GPP_US_NAT` keeps its name too — the name is historical and does not narrow what is read.
- **A CMP answer is reported even when it says GDPR does not apply** — a TC string is adopted whenever your CMP publishes a complete set, not only when `IABTCF_gdprApplies` is 1. That value still decides gating; it no longer decides whether the answer is worth reporting. Snapshots and events now carry what the CMP said about users outside GDPR scope.
- **IAB consent is read only where the specs put it** — the SDK reads the preferences file the IAB in-app specs require a CMP to write, plus a few known fallbacks, instead of searching every preferences file the app has. A CMP that writes elsewhere is named with `ConsentOptions.prefsProvider`.
- **Four `ConsentOptions` fields are deprecated no-ops** — `preferTcfWhenBothPresent`, `gppUsNatSid`, `allowPrefsScan` and `enableUsNatMapping`. They kept their place so existing code still compiles; TCF now always owns the European axis when your CMP publishes one, and GPP sections are never filtered by id.
- **`enableTCFDataCollection` documents its real default** — reading IAB TCF consent data is off until you call it with `true`. The behaviour is unchanged; the documentation said the opposite.
- **The SDK no longer forces itself whole into your shrunk build** — the rules that arrive with the dependency used to keep every class and every member under `com.adshift.sdk`, private ones included, so R8 could neither drop the parts of the SDK your app never reaches nor rename anything. They now keep the documented API and the models that go on the wire, and hand the rest to the shrinker. In our sample app the SDK's share of the DEX fell from 217 KB to 48 KB, and from 2,174 methods to 485. Two consequences are worth knowing. Stack traces from inside the SDK now arrive obfuscated, so upload the `mapping.txt` your release build produces — Crashlytics and the Play Console do this for you, and your public API calls are unaffected either way. And if you ever copied `-keep class com.adshift.sdk.** { *; }` into your own configuration, as an earlier version of our documentation asked you to, delete it: kept by hand it reinstates the old size in full.

### Removed
- **Types that were never meant to be callable are no longer public** — the SDK now carries a machine-readable record of its public API, and setting it up showed that a third of that surface was public by accident rather than by design. Withdrawing it is a breaking change in name only: nothing removed here appears in any documented signature, and the compiler will tell you at once if you had reached for it.
  - `ASLogger`'s logging methods (`debug`, `info`, `warn`, `error`, `verbose`, `Assert` and its own `setLogLevel`) are internal. If you were logging through them, use `android.util.Log` or your own logger. `ASLogger.LogLevel` stays public, because `AdShiftLib.setLogLevel` takes it, and that call is unchanged.
  - The AIDL interfaces the SDK uses to ask Chinese device vendors for an OAID — 26 classes under a nested `repeackage` package — are package-private and sit next to the code that uses them. OAID collection itself is unchanged: the identity these interfaces bind to comes from the vendor, not from where we keep our copy.
- **Twenty `-keep` rules you were told to copy are gone, because they never did anything** — the README asked you to add rules for `repeackage.com.*`, naming a top-level package that does not exist in this SDK. Our copies of those vendor interfaces have always lived under `com.adshift.sdk`, and they need no rules of their own: what makes a vendor's OAID service bind is the descriptor string the vendor published, and a string literal survives both shrinking and renaming. If you added the block by hand, deleting it changes nothing. If you kept it, it stays harmless.

### Fixed
- **Release builds with code shrinking start again** — the SDK now ships the shrinker rules its JSON layer needs, so an app built with `minifyEnabled true` no longer stops at startup with `TypeToken must be created with a type argument`. The rules arrive with the dependency; nothing goes into your own `proguard-rules.pro`. Affects 3.0.0 only.
- **A GPP string no longer displaces a TC string your CMP still publishes** — the two axes are independent, and a US signal never takes over the European one.
- **A denial outlives the CMP signal that expressed it** — when a CMP's keys disappear, a consent it produced is forgotten, but a refusal is not: the advertising identifier stays gated instead of being handed back because the keys went away.
- **Reserved values in `IABGPP_GppSID` are no longer read as sections** — `-1` ("no section applies") and `0` ("not yet determined") were taken for section ids.
- **`IABGPP_GppSID` stored as a number is read** — a CMP writing it as an integer rather than a string was ignored, and a store holding any IAB key with an unexpected type no longer fails the whole refresh.

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
