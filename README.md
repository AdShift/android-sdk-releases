# AdShift Android SDK

[![Maven Central](https://img.shields.io/maven-central/v/com.adshift/android-sdk)](https://central.sonatype.com/artifact/com.adshift/android-sdk)
[![Platform](https://img.shields.io/badge/platform-Android%205.0%2B-blue.svg)](https://developer.android.com/)
[![License](https://img.shields.io/badge/license-Proprietary-lightgrey.svg)](LICENSE)

The AdShift Android SDK measures installs and in-app events, resolves direct and deferred deep links, and carries user consent signals to ad partners.

This repository is the public home for the SDK's release notes and version history. The SDK is proprietary: the source is not published here and the artifact is distributed through Maven Central.

## Release notes

- Changelog: [CHANGELOG.md](CHANGELOG.md)
- All releases: https://github.com/AdShift/android-sdk-releases/releases

## Installation

```kotlin
dependencies {
    implementation("com.adshift:android-sdk:3.0.0")
}
```

Repositories, permissions and the Play Services dependency are covered in the [installation guide](https://dev.adshift.com/docs/android-sdk/installation).

## Documentation

The full documentation lives at [dev.adshift.com](https://dev.adshift.com/docs/android-sdk):

| | |
|---|---|
| [Quickstart](https://dev.adshift.com/docs/android-sdk/quickstart) | Minimal integration, start to finish |
| [Integration](https://dev.adshift.com/docs/android-sdk/integration) | Initialization, lifecycle and configuration |
| [In-app events](https://dev.adshift.com/docs/android-sdk/events) | Event tracking and revenue |
| [Deep links](https://dev.adshift.com/docs/android-sdk/deeplinks) | Direct, deferred and RightLink handling |
| [Consent](https://dev.adshift.com/docs/android-sdk/consent) · [DMA](https://dev.adshift.com/docs/android-sdk/dma) | GDPR, TCF 2.2 and Google consent signals |
| [Backup rules](https://dev.adshift.com/docs/android-sdk/backup) | Auto Backup exclusions and manifest conflicts |
| [Push notifications](https://dev.adshift.com/docs/android-sdk/push-notifications) | Attributing push-driven re-engagement |
| [Troubleshooting](https://dev.adshift.com/docs/android-sdk/troubleshooting) · [Debugging](https://dev.adshift.com/docs/android-sdk/debugging) | Verifying an integration |

## Requirements

- Android 5.0 (API 21) or newer
- Compiled against API 35

## Support

- Email: support@adshift.com
- Documentation: https://dev.adshift.com
- Issues are disabled here; please contact support.

## License

Copyright © 2026 AdShift sp. z o.o. All rights reserved.

This SDK is proprietary software. See [LICENSE](LICENSE) for details.
