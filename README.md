# PuerixSDK — Android

Native Android SDK for age verification with liveness detection and document capture.

## Installation

Add the Puerix Maven repository and dependency to your `build.gradle`:

```gradle
// settings.gradle or root build.gradle
repositories {
    google()
    mavenCentral()
    maven { url 'https://raw.githubusercontent.com/Puerix/puerix-sdk-android/main/maven-repo' }
}
```

```gradle
// app/build.gradle
dependencies {
    implementation 'com.puerix:puerix-sdk:0.1.0'
}
```

## Usage

```kotlin
// 1. Initialize
PuerixSDK.initialize(PuerixConfig(apiKey = "YOUR_API_KEY"))

// 2. Start verification
PuerixSDK.startVerification(
    activity = this,
    requestCode = 1234,
    subject = "user-123",
    ageLimit = 18,
)

// 3. Handle result
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    super.onActivityResult(requestCode, resultCode, data)
    PuerixSDK.handleVerificationResult(requestCode, resultCode, data, 1234) { result ->
        if (result.isApproved) {
            Log.d("Puerix", "Approved! Session: ${result.sessionId}")
        }
    }
}
```

## Requirements

- Android API 21+
- API key from [puerix.com](https://puerix.com)

## License

Copyright (c) 2024 Puerix. All rights reserved. Proprietary license.
