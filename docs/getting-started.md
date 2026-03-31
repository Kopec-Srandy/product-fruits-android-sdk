# Getting started with the Product Fruits Android SDK

This guide walks through adding `com.productfruits:sdk` to an Android app and wiring identity, screens, and push.

## Prerequisites

- **minSdk 24** (as published in the AAR)
- **JDK 17**, **Android Studio** with a recent AGP
- Product Fruits account — [my.productfruits.com](https://my.productfruits.com)

## Step 1: Add the Maven repository and dependency

In **Gradle** (Kotlin DSL), add the GitHub-hosted Maven repo and the SDK. Use a **fixed version** (see [Releases](https://github.com/product-fruits/product-fruits-android-sdk/releases)).

**`settings.gradle.kts`** (or root `dependencyResolutionManagement`):

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven {
            url = uri("https://raw.githubusercontent.com/product-fruits/product-fruits-android-sdk/main/maven-repo")
            content { includeGroup("com.productfruits") }
        }
    }
}
```

**App `build.gradle.kts`:**

```kotlin
dependencies {
    implementation("com.productfruits:sdk:1.0.5") // replace with your version
}
```

## Step 2: Credentials from the dashboard

1. Open [Product Fruits](https://app.productfruits.com).
2. Go to **Settings → Mobile Apps**.
3. Copy **Project code** and **Application ID** (UUID for this Android app).

`applicationId` in Gradle (`com.yourcompany.app`) is **not** the same as Product Fruits **Application ID**.

## Step 3: Initialize the SDK

Create **`ProductFruitsConfig`**, then **`ProductFruits`**. Keep a strong reference (e.g. in `Application`, a `ViewModel`, or a DI singleton) for the app lifetime.

**`Application` subclass (recommended):**

```kotlin
class MyApp : Application() {
    lateinit var productFruits: ProductFruits
        private set

    override fun onCreate() {
        super.onCreate()
        ProductFruits.ensurePushNotificationChannel(this)

        val config = ProductFruitsConfig(
            projectCode = "your-project-code",
            applicationID = "your-mobile-app-uuid",
            language = "en"
        ).logging(true)

        productFruits = ProductFruits(this, config)
    }
}
```

Register the class in **`AndroidManifest.xml`**:

```xml
<application
    android:name=".MyApp"
    ...>
```

**Staging / custom host:**

```kotlin
import java.net.URL

val config = ProductFruitsConfig(...)
    .setEnvironmentUrl(URL("https://your-staging-host"))
// Self-signed local dev only:
// .trustAllCertificatesForLocalDev(true)
```

## Step 4: Identify users

Call **`identify`** after you know the signed-in user (same idea as iOS).

```kotlin
productFruits.identify("user-123") { success, error ->
    if (!success) Log.e("PF", "identify failed", error)
}
```

With optional profile fields:

```kotlin
productFruits.identify(
    username = "john@example.com",
    firstName = "John",
    lastName = "Doe",
    role = "admin",
    additionalProperties = mapOf(
        "plan" to "premium",
        "department" to "engineering"
    )
) { success, error -> /* ... */ }
```

**Anonymous session** (optional):

```kotlin
productFruits.anonymous { success, error -> /* ... */ }
```

**Logout:**

```kotlin
productFruits.reset()
```

## Step 5: Screens and events

**Screen** (for targeting and `productfruits:screen_view` when the user is identified):

```kotlin
productFruits.setScreen("home")

// Clear when leaving a flow, if needed:
productFruits.setScreen(null)
```

Call from `onResume` of each screen, or your navigation callback.

**Custom events** (requires an identified user — not anonymous-only flows that never called `identify`):

```kotlin
productFruits.track(
    name = "purchase_completed",
    properties = mapOf(
        "item_id" to "123",
        "amount" to 29.99,
        "currency" to "USD"
    )
) { success, error -> /* ... */ }
```

## Step 6: Show an experience manually

```kotlin
productFruits.show("YOUR-EXPERIENCE-GUID") { success, error ->
    if (!success) Log.e("PF", "show failed", error)
}
```

## Step 7: Push notifications (FCM)

1. Add Firebase to the app (`google-services.json`, Google Services Gradle plugin).
2. Dependencies (example):

```kotlin
implementation(platform("com.google.firebase:firebase-bom:34.10.0"))
implementation("com.google.firebase:firebase-messaging")
```

3. **`AndroidManifest.xml`** — service and default channel id:

```xml
<meta-data
    android:name="com.google.firebase.messaging.default_notification_channel_id"
    android:value="your_channel_id" />

<service
    android:name="com.productfruits.sdk.ProductFruitsFirebaseMessagingService"
    android:exported="false">
    <intent-filter>
        <action android:name="com.google.firebase.MESSAGING_EVENT" />
    </intent-filter>
</service>
```

4. **`Application.onCreate`:** `ProductFruits.ensurePushNotificationChannel(this)` (already shown above).

5. **Android 13+:** request **`POST_NOTIFICATIONS`**; then obtain the FCM token and register:

```kotlin
FirebaseMessaging.getInstance().token.addOnSuccessListener { token ->
    productFruits.updatePushToken(token)
    productFruits.refreshPushNotificationState()
}
```

6. **Launcher activity** — `singleTop` (or appropriate) and notification tap handling:

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    ProductFruits.handleNotificationIntent(this, intent)
    // ...
}

override fun onNewIntent(intent: Intent) {
    super.onNewIntent(intent)
    setIntent(intent)
    ProductFruits.handleNotificationIntent(this, intent)
}
```

## Step 8: Test and debug

- Enable **`config.logging(true)`** and filter Logcat for **`ProductFruits`**.
- **`productFruits.version()`** / **`ProductFruits.VERSION`** — SDK version string.

## Next steps

- Configure mobile experiences in the Product Fruits workspace.
- See [examples.md](examples.md) and [api-reference.md](api-reference.md).

## Troubleshooting

| Issue | What to check |
|--------|----------------|
| Gradle cannot resolve `com.productfruits:sdk` | Maven `url` in `settings.gradle.kts`, correct version, network |
| `identify` / `track` / `show` no-ops | User must be **identified** (`identify`); check `getCurrentState().isActive` |
| Push not visible | **POST_NOTIFICATIONS** (API 33+), channel created (`ensurePushNotificationChannel`), token sent after `identify` |
| Tap does not open content | `handleNotificationIntent` in launcher activity; `applicationID` in config matches the app in the dashboard |

Support: [support@productfruits.com](mailto:support@productfruits.com).
