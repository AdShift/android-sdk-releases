# Changelog

All notable changes to the AdShift Android SDK will be documented in this file.

## [4.0.0] - 2026-10-29

Major release of deep linking and link attribution: one rule decides which links are RightLink links, a link's click is sent only after `start()`, and direct and deferred links come back in one result shape. Most changes are in behaviour rather than in the API, and some change what your deep link listener receives without a build error, so upgrading requires code changes. See the upgrade notes below.

### Changed
- **Only RightLink links come back `FOUND`** — a link on `rightlink.me` or its subdomains, or on a branded domain you declared with `setBrandedDomains`, is `FOUND`, also when it arrives wrapped in your app's scheme (see Added). Any other link, your app's own scheme and web links included, comes back `NOT_FOUND` with `uri` only: `queryParams` is empty and `deepLinkValue` is null. In 3.2 every link came back `FOUND` with its query parameters. So an app that routes its own links through the deep link listener now reads them from `uri`, and an app that navigates on `FOUND` ignores the links of a branded domain it did not declare. A Meta re-engagement link also comes back `NOT_FOUND`, and is still attributed.
- **Every RightLink link counts as a click** — a link on a RightLink or declared branded domain now sends a click, and the open it starts carries it, also when the link has no `pid` and `c`. 3.2 attributed only links that carried both, so opens from such links can now be attributed.
- **A link's click is sent after `start()`, with the consent of that moment** — the click is recorded when the link is opened, which may be before `start()`, and sent only after it, with the advertising ID and the consent read when it goes out. An app that asks for consent before it starts the SDK no longer sends a click ahead of the user's answer.
- **Only the link parameters AdShift uses leave the device** — of a RightLink link's query, only attribution, campaign, deep link and ad click ID parameters are sent: `pid`, `c`, `campaign`, `is_retargeting`, `clickid`, `deep_link_value`, `deep_link_sub1` to `deep_link_sub5`, `gclid`, `gbraid`, `wbraid`, `ttclid`, `fbclid`, and parameters starting with `as_` or `utm_`. Anything else, such as an email address or a user ID in a campaign link, stays in `DeepLinkResult.queryParams` for your own routing and is never sent, so it does not appear in reports or postbacks either.
- **An open by a link reports one `app_open`, which carries the click** — after `start()` the SDK waits briefly for the link your first Activity hands over, so the open's one `app_open` (or, on a first launch, the install) carries its click, and the events you track meanwhile follow it. On a cold start, events are therefore sent about a second after `start()` at the earliest. A short link opened without a network is attributed to that open once the network is back.
- **`initSdk` makes no network call** — the developer key is checked at the first `start()`, so an app that shows a consent screen between the two calls reaches the network only after it. `start()` records the open at once, offline too, and nothing is sent until the key is accepted; its listener gets `onSuccess` then, or `onError` for a rejected key.
- **The deferred deep link is answered within 5 seconds of `start()`** — with the answer, or with `ERROR` (`NETWORK` without a network, `TIMEOUT` otherwise), also on a first launch without a network, which in 3.2 got no answer at all. A link the user opened wins: the deferred answer waits briefly after `start()` for a link your first Activity hands over, and is not delivered once the user has opened a link in the same process. An answer that arrives after the 5 seconds is handed over at the next start, cold or from the background. While the lookup is still unanswered, each return from the background is a new try with its own 5-second answer, so your listener can receive `ERROR` for it more than once.
- **An unanswered deferred lookup is asked again** — a start whose lookup gets no answer asks again at the next start, a return from the background included, on up to three starts; a start without a network does not count. The lookup sends the Play Install Referrer and, under the consent rules that already apply to them, the advertising ID or the OAID. Without any of them the SDK asks nothing and the answer is `NOT_FOUND`.
- **Up to five results wait for a listener** — results that arrive while no listener is set are delivered in order, up to five, once one is set; 3.x kept only the last one. The listener may be set before `initSdk`, and `setDeepLinkListener(null)` clears it.
- **A short link that cannot be resolved is tried again** — each failed try gives `ERROR`, and the SDK tries the link again at the next `start()` or when the network returns, up to three tries within a day. A try that resolves it delivers `FOUND` then, which can be at a later start; after the last failed try the app gets `NOT_FOUND`. A deleted short link is `NOT_FOUND`. Short codes are read in either case, so a link from an upper-case QR code resolves. Parameters appended to a short link no longer override the ones stored with it.
- **Meta re-engagement is read only from the Meta apps** — `al_applink_data` counts only as the Bundle that Facebook, Instagram, Messenger or Facebook Lite delivers when it launched the Activity. The payload in a URL's query or in a String extra, and a `campaign_ids` URL parameter, are ignored, so only the Meta apps can report a Meta re-engagement.
- **`handleAppLinkIntent` and `setDeepLinkListener` no longer throw before `initSdk`** — `handleAppLinkIntent` logs an integration error and ignores the intent, and `setDeepLinkListener` keeps the listener for when the SDK is initialized. An intent that is not `ACTION_VIEW` with data is ignored with a log, and a secondary process of your app takes no link in.

### Added
- **`DeepLinkResult.deepLinkValue`, `source` and `clickId`** — `deepLinkValue` is what to navigate on: `deep_link_value` of the link or of the deferred answer. `source` says how the link reached the app (`APP_LINK`, `SCHEME`, `CTA` for the button of a RightLink landing page, `PUSH`, `META_APPLINK`, `DEFERRED`, or `FOREIGN` for a web link of another domain). `clickId` names the click the open was attributed to. `sub1` to `sub5` are now filled for links the app was opened with too. `uri` is the link as opened, unwrapped from your scheme and with a short link expanded; on a deferred result it is `deepLinkValue` as a Uri, as in 3.x.
- **`handleAppLinkIntent(intent, savedInstanceState)`** — called from `onCreate`, it tells an Activity the system re-created (a rotation, a return after the process was killed) from a new tap, so a re-created Activity sends no second click and gets no second delivery. The SDK also takes one tap once in other cases: the same Intent again within 10 seconds, an Intent relaunched from Recents, and the landing page's button reopening the app for a click that already opened it.
- **RightLink links inside your app's scheme** — a link in your scheme that carries a RightLink link in `as_rl`, such as `<scheme>://rightlink?as_rl=<encoded link>`, is read as the link it wraps, short links included, and attributed. 3.x treated it as a foreign link.
- **Test traffic** — `AdShiftLib.setTestMode(true)`, called before `start()`, marks every click and event as test traffic, which is kept out of your reports. The deferred deep link lookup is not marked, and a debug build does not turn it on by itself. On Google's test devices (Firebase Test Lab, which also runs the Google Play pre-launch report) the SDK marks its traffic as test traffic by itself, so the devices Google uses to check an uploaded build do not show up as your users.

### Removed
- **AppCompat and Material are no longer pulled into your app** — the SDK never used `androidx.appcompat` or `com.google.android.material`, but declared them, so every app received them through it. If your app uses either without declaring it, for example in a theme or a layout, declare it now.
- **Huawei's `ads-installreferrer` is no longer needed** — the SDK has never read the AppGallery referrer; from Huawei it reads only the OAID through `ads-identifier`. Earlier versions of our instructions asked you to add the referrer library; remove it unless your app uses it for something else.

### Fixed
- **A deferred answer that lost to an opened link no longer opens later** — a deferred answer of a launch in which the user opened a link was kept and could navigate the app at a later start from the icon, without the user doing anything. It is now final, whether it arrived before or after the link.
- **A tap's attribution no longer reaches a much later open** — when the `app_open` of a link's open was never sent, for example because the process died before `start()`, the link's attribution was kept for the next open, which could be a session started from the icon days later. It now expires 10 minutes after the tap.
- **An install restored from another install's backup is measured as its own** — an app that replaced the SDK's backup rules could restore the SDK state of another install, which skipped this install's `app_install` and its deferred lookup. State that belongs to another install is now cleared in `initSdk`.
- **`isFirstSession` is right for a link handed over before `start()`** — in 3.2 such a link always read `true`; it now says whether this is the first run of the SDK on this install.
- **Internationalized branded domains match** — a branded domain is compared in punycode, whichever form your app or the link uses.

### Upgrading

Pass `savedInstanceState` to `handleAppLinkIntent` in `onCreate`. `onNewIntent` keeps the one-argument call:

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    // before: AdShiftLib.handleAppLinkIntent(intent)
    AdShiftLib.handleAppLinkIntent(intent, savedInstanceState)
}

override fun onNewIntent(intent: Intent) {
    super.onNewIntent(intent)
    AdShiftLib.handleAppLinkIntent(intent)
}
```

The one-argument call still compiles in `onCreate`, but a re-created Activity then counts as a new tap once the 10-second window has passed.

Declare every branded link domain after `initSdk` and before `start()`, or its links come back `NOT_FOUND`:

```kotlin
AdShiftLib.setBrandedDomains(listOf("go.yourbrand.com"))
```

Navigate on `FOUND` only, with `deepLinkValue` (a campaign can also route by `sub1` to `sub5` alone), and validate the value first, since anyone can write a link. Read your app's own links from `uri` on `NOT_FOUND`: `queryParams` is filled for RightLink links only. Treat `ERROR` as "no deep link for now": a first screen waiting for the deferred deep link gets an answer within 5 seconds of `start()`, and a short link that failed can still arrive as `FOUND` at a later start.

```kotlin
AdShiftLib.setDeepLinkListener(object : DeepLinkListener {
    override fun onDeepLinking(result: DeepLinkResult) {
        when (result.status) {
            DeepLinkStatus.FOUND -> { /* validate result.deepLinkValue, then navigate */ }
            DeepLinkStatus.NOT_FOUND -> { /* your own links: route on result.uri */ }
            DeepLinkStatus.ERROR -> { /* continue without a deep link */ }
        }
    }
})
```

Register the listener once, in `Application.onCreate`. A listener set in an Activity holds that Activity until it is replaced, so clear it there with `AdShiftLib.setDeepLinkListener(null)` in `onDestroy`.

If your app uses AppCompat or Material without declaring them, add them to your dependencies.

Declare the Activity that handles links with `android:launchMode="singleTask"` and handle the link in `onNewIntent`. With the default launch mode, Android can bring a task opened by a link back to the front without delivering a second tap of the same link, and the SDK cannot see a tap the system does not deliver.

`DeepLinkResult`'s constructor and `copy` take three more parameters. Kotlin code that builds a `DeepLinkResult` itself, such as a test, needs a recompile; Java code passes the three new arguments.

## [3.2.0] - 2026-10-06

### Added
- **Google's App Set ID, when your app includes Google's library** — add `com.google.android.gms:play-services-appset` and the SDK reads the [App Set ID](https://developer.android.com/identity/app-set-id) and sends it with your events. It follows the same consent rule as the advertising ID, but the user's opt-out of ads personalisation does not stop it. It is not read on devices without Google Play services 21.28.00 or later, or once you remove the `AD_ID` permission the SDK adds to your manifest, which stops the advertising ID too. To turn off only the App Set ID, call `AdShiftLib.setCollectAppSetId(false)` on every app start, after `initSdk()` and before `start()`. If your app already includes the library, for example through Google Mobile Ads, it starts sending the ID with this version. Before you enable it, update your Data safety form ("Device or other IDs") and your privacy policy.

### Fixed
- **Excluding Google's advertising ID library no longer breaks the app** — excluding `play-services-ads-identifier` (or `play-services-basement`) could fail the release build in R8 with "Missing class", or crash `AdShiftLib.initSdk` with `NoClassDefFoundError`. The SDK now treats the missing library as Google Play services being unavailable and logs a warning. To stop collecting the advertising ID, remove the `AD_ID` permission instead: the SDK adds it to your manifest, and once you remove it with `tools:node="remove"` the SDK reads neither the advertising ID nor the App Set ID, whatever your target SDK.
- **Events no longer reach AdShift before the install** — on a first launch the install waits for the Play referrer, and an event tracked meanwhile, right after `start()`, used to be delivered first, before AdShift knew of the install. Event delivery now waits until the install is queued, and the install goes first, so on a first launch your events leave after the install, which waits until the Play Install Referrer has answered or the SDK has stopped waiting for it. If the SDK cannot read the referrer and leaves the install to the next launch, the held events are sent without it. Server-to-server clicks are not held, and launches after the install is queued are not affected.
- **A `stop()` and `start()` during the first launch send one install** — calling them while the install was still being prepared, as an app that ties them to its activity's lifecycle can, sent the install twice.
- **An install is no longer lost when the app is killed during its first launch** — a kill right after the install was queued could lose it for good. The install is now saved before it counts as sent, and the next launch sends it, once.
- **`start()` and `stop()` no longer crash when Android refuses the network callback** — registering it past the per-app limit on Android 11 and later, or unregistering it, could throw out of `start()` or `stop()`. The SDK now carries on without it.

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
