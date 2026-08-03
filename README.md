# AdShift Android SDK

[![Maven Central](https://img.shields.io/maven-central/v/com.adshift/android-sdk)](https://central.sonatype.com/artifact/com.adshift/android-sdk)
[![Platform](https://img.shields.io/badge/platform-Android%205.0%2B-blue.svg)](https://developer.android.com/)
[![License](https://img.shields.io/badge/license-Proprietary-lightgrey.svg)](LICENSE)

The AdShift Android SDK provides install tracking, deep linking, in-app event attribution and consent handling.

This repository carries the release notes and the integration reference. The SDK is proprietary and distributed through Maven Central — there is no source code here.

## Latest version

- Release notes: [CHANGELOG.md](CHANGELOG.md)
- All releases: https://github.com/AdShift/android-sdk-releases/releases
- Artifact: https://central.sonatype.com/artifact/com.adshift/android-sdk

## Installation

```kotlin
dependencies {
    implementation("com.adshift:android-sdk:3.0.0")
}
```

Apps distributed through Google Play also need the advertising ID library — see the [integration guide](https://dev.adshift.com/docs/android-sdk) for the current dependency and the manifest permissions.

## Quick start

Initialize the SDK in your `Application` class, so it is running before any deferred deep link arrives:

```kotlin
import com.adshift.sdk.core.AdShiftLib

class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()

        AdShiftLib.initSdk(
            context = this,
            devKey = "your-dev-key"
        )
    }
}
```

Then start tracking:

```kotlin
AdShiftLib.start()
```

If you gate tracking on a consent dialog, call `start()` once the user has answered — events tracked before that are queued, not lost.

## Documentation

- Main documentation: https://dev.adshift.com/
- Android SDK integration guide: https://dev.adshift.com/docs/android-sdk
- Deep linking guide: https://dev.adshift.com/docs/deeplinks-rightlink

## Requirements

- Android 5.0 (API 21) or newer
- Compiled against API 35

## Privacy

The SDK supports GDPR and TCF 2.2 consent, Google consent signals (`ad_storage`, `ad_user_data`, `ad_personalization`) and Limit Ad Tracking enforcement. Advertising identifiers are collected only when consent allows it.

## Support

- Email: support@adshift.com
- Documentation: https://dev.adshift.com
- Issues are disabled here; please contact support.

## License

Copyright © 2026 AdShift sp. z o.o. All rights reserved.

This SDK is proprietary software. See [LICENSE](LICENSE) for details.
