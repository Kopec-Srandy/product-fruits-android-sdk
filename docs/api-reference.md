# API reference — Product Fruits Android SDK

Kotlin API for **`com.productfruits:sdk`**. Callbacks are invoked on background threads unless noted; run UI updates on the **main thread**.

## Table of contents

- [ProductFruitsConfig](#productfruitsconfig)
- [ProductFruits (instance)](#productfruits-instance)
- [ProductFruits (companion)](#productfruits-companion)
- [Push](#push)
- [State & errors](#state--errors)

---

## ProductFruitsConfig

```kotlin
class ProductFruitsConfig(
    val projectCode: String,
    val applicationID: String,
    val language: String
)
```

| Parameter | Description |
|-----------|-------------|
| `projectCode` | Workspace project code from the dashboard |
| `applicationID` | Mobile app UUID (**Settings → Mobile Apps**), not the Gradle `applicationId` |
| `language` | Default language (e.g. `"en"`) |

### Builder-style methods (chainable)

| Method | Description |
|--------|-------------|
| `logging(Boolean)` | Verbose SDK logging |
| `apiHost(URL)` | Override API base URL |
| `setEnvironmentUrl(URL)` | Sets both API and settings host (staging) |
| `trustAllCertificatesForLocalDev(Boolean)` | **Debug only** — self-signed HTTPS (e.g. emulator → `10.0.2.2`) |
| `deviceProperties(Map<String, Any?>)` | Extra device / segment properties |
| `setUserLanguage(String)` | Updates `_language` in device properties |
| `anonymousIDFactory(() -> String)` | Custom anonymous id generator |
| `activityStorageMaxSize(UInt)` | Activity queue size cap |
| `activityStorageMaxAge(UInt?)` | Max age of stored activities (seconds) |

Constants: **`ProductFruitsConfig.DEFAULT_API_HOST`** — `https://app.productfruits.com`

---

## ProductFruits (instance)

```kotlin
class ProductFruits(
    context: Context,
    config: ProductFruitsConfig
)
```

Use **`context.getApplicationContext()`**-safe usage; the constructor stores application context internally for network and storage.

### Identity

| Method | Description |
|--------|-------------|
| `identify(username, firstName?, lastName?, signUpAt?, role?, additionalProperties?, completion?)` | Registers the user; may fetch and show qualified experiences |
| `anonymous(completion?)` | Switches to anonymous mode and posts site config |
| `reset()` | Clears user session (logout) |

### Screen & analytics

| Method | Description |
|--------|-------------|
| `setScreen(screenName: String?)` | Current screen for targeting; emits `productfruits:screen_view` when name changes and user is active |
| `getCurrentScreen(): String?` | Last non-empty screen passed to `setScreen` |
| `track(name, properties?, completion?)` | Custom event (**requires** identified user) |
| `screen(title, properties?)` | Low-level screen event (used internally when `setScreen` changes) |

### Experiences

| Method | Description |
|--------|-------------|
| `show(experienceID, completion?)` | Loads tool content and presents announcement UI |
| `showRaw(jsonString, launchContext?, completion?)` | Parses JSON descriptor; pass an **Activity** as `launchContext` when possible so modals appear above your UI |

### Qualification

| Method | Description |
|--------|-------------|
| `qualify(completion: (Result<ToolQualificationResponse>) -> Unit)` | GET qualify endpoint |
| `getQualifiedTools(completion: (Result<List<String>>) -> Unit)` | Convenience over `qualify` |

**`ToolQualificationResponse`:** `userId`, `qualifiedTools`, `qualifiedSegments`, `groupId` (nullable fields).

### Feedback & health

| Method | Description |
|--------|-------------|
| `submitFeedback(message, email?, screenshotURL?, metadata?, completion?)` | Posts feedback |
| `checkHealth(completion: (Boolean, Throwable?) -> Unit)` | Simple connectivity check |

### Push (FCM)

| Method | Description |
|--------|-------------|
| `updatePushToken(token: String)` | Registers FCM token; triggers `device_updated` when user + token ready |
| `refreshPushNotificationState()` | Re-reads notification permission and sends updated device state |

### State & version

| Method | Description |
|--------|-------------|
| `getCurrentState(): ProductFruitsState` | Snapshot of version, user, device, session, `isActive` |
| `version(): String` | Same as **`ProductFruits.VERSION`** |
| `getConfigLanguage(): String` | Config language string |

**`isActive`:** `true` when a non-empty user id is stored (after `identify` or `anonymous`).

---

## ProductFruits (companion)

| Method | Description |
|--------|-------------|
| `ensurePushNotificationChannel(context: Context)` | Creates high-importance channel for `default_notification_channel_id` — call from **`Application.onCreate`** |
| `handleNotificationIntent(context: Context, intent: Intent?): Boolean` | Parses notification tap; call from launcher **`onCreate`** / **`onNewIntent`** |
| `VERSION: String` | SDK version constant |

---

## Push

- Implement **`com.productfruits.sdk.ProductFruitsFirebaseMessagingService`** in the manifest (FCM `MESSAGING_EVENT`).
- The SDK posts **local notifications in the foreground** and relies on the system + channel for **background** notification payloads.
- **Android 13+:** request **`POST_NOTIFICATIONS`** before expecting visible notifications.

There is **no** Android equivalent of iOS `getPushToken()` on the public API — use **`FirebaseMessaging.getInstance().token`** and **`updatePushToken`**.

**Deep links / URL routing:** handled via notification intent extras and **`handleNotificationIntent`**; there is no separate `didHandleURL` API on Android.

---

## State & errors

### ProductFruitsState

| Property | Type | Description |
|----------|------|-------------|
| `version` | `String` | SDK version |
| `projectCode` | `String` | From config |
| `userID` | `String` | Current user id or anonymous id |
| `isAnonymous` | `Boolean` | Anonymous mode |
| `deviceID` | `String` | Stable device id used by the SDK |
| `sessionID` | `String?` | Session id |
| `isActive` | `Boolean` | User identified or anonymous session established |
| `lastUpdated` | `String` | ISO timestamp of snapshot |
| `status` | `IdentificationStatus` | `NOT_IDENTIFIED`, `ANONYMOUS`, `IDENTIFIED` |
| `statusDescription` | `String` | Human-readable status |

**`toDictionary()`** — export as `Map` (e.g. for debugging).

### ProductFruitsError & ProductFruitsException

`ProductFruitsException` wraps **`ProductFruitsError`**:

- `UserNotIdentified`
- `InvalidUsername`
- `InvalidToolId`
- `InvalidConfiguration`
- `InvalidResponse`
- `NetworkError`
- `SdkNotAvailable`
- `ActivityProcessingFailed`
- `InternalError`

---

## Threading & lifecycle

- Hold **one** `ProductFruits` instance for the process (e.g. `Application` or DI scope).
- Completion lambdas may run off the main thread; use **`runOnUiThread`**, **`Handler(Looper.getMainLooper())`**, or coroutine **`withContext(Dispatchers.Main)`** for UI.
- **No** `NotificationCenter` / `productFruitsStateChanged` broadcast — poll **`getCurrentState()`** or observe your own layer after `identify` / `reset`.

---

## iOS parity notes

| iOS | Android |
|-----|---------|
| `ProductFruits.Config` | `ProductFruitsConfig` |
| `ProductFruits(config:)` with UIKit context | `ProductFruits(context, config)` |
| `setPushToken(Data)` | `updatePushToken(String)` (FCM string) |
| `didReceiveNotification` | `ProductFruitsFirebaseMessagingService` + `handleNotificationIntent` |
| `trustLocalhostCertificates` | `trustAllCertificatesForLocalDev` |

For the latest behavior details, refer to the source in the private SDK repo or published sources jar when available.
