# Examples — Product Fruits Android SDK

Practical patterns for Kotlin apps. Adjust package names and DI to match your project.

## 1. Application singleton

```kotlin
// MyApp.kt
class MyApp : Application() {
    lateinit var pf: ProductFruits
        private set

    override fun onCreate() {
        super.onCreate()
        ProductFruits.ensurePushNotificationChannel(this)
        val config = ProductFruitsConfig(
            projectCode = BuildConfig.PF_PROJECT_CODE,
            applicationID = BuildConfig.PF_APPLICATION_ID,
            language = resources.configuration.locales[0].language.ifBlank { "en" }
        ).logging(BuildConfig.DEBUG)
        pf = ProductFruits(this, config)
    }
}
```

Store secrets in **`BuildConfig`** via `build.gradle.kts` `buildTypes` / `defaultConfig`, or inject from remote config — avoid hard-coding in source for release builds.

## 2. Login / logout

```kotlin
fun onLogin(userId: String) {
    (application as MyApp).pf.identify(userId) { ok, err ->
        if (ok) registerFcmToken()
    }
}

fun onLogout() {
    (application as MyApp).pf.reset()
}

private fun registerFcmToken() {
    FirebaseMessaging.getInstance().token.addOnSuccessListener { token ->
        val pf = (application as MyApp).pf
        pf.updatePushToken(token)
        pf.refreshPushNotificationState()
    }
}
```

## 3. Jetpack Navigation / one Activity

Update the current screen when the destination changes:

```kotlin
navController.addOnDestinationChangedListener { _, destination, _ ->
    val label = destination.label?.toString() ?: destination.route ?: return@addOnDestinationChangedListener
    (application as MyApp).pf.setScreen(label)
}
```

## 4. Compose + single Activity

```kotlin
@Composable
fun MyAppNav() {
    val navController = rememberNavController()
    val pf = (LocalContext.current.applicationContext as MyApp).pf

    LaunchedEffect(navController) {
        navController.addOnDestinationChangedListener { _, dest, _ ->
            dest.route?.let { pf.setScreen(it) }
        }
    }
    NavHost(navController, startDestination = "home") { /* ... */ }
}
```

## 5. Request notification permission (API 33+)

```kotlin
val permissionLauncher = rememberLauncherForActivityResult(
    ActivityResultContracts.RequestPermission()
) { granted ->
    if (granted) registerFcmToken()
}

LaunchedEffect(Unit) {
    if (Build.VERSION.SDK_INT >= 33 &&
        ContextCompat.checkSelfPermission(context, Manifest.permission.POST_NOTIFICATIONS)
        != PackageManager.PERMISSION_GRANTED
    ) {
        permissionLauncher.launch(Manifest.permission.POST_NOTIFICATIONS)
    } else {
        registerFcmToken()
    }
}
```

## 6. Manual experience + preview JSON

```kotlin
// By ID from the dashboard
pf.show("aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee") { ok, err -> }

// Raw JSON (e.g. preview) — pass an Activity context so the modal can show on top
pf.showRaw(jsonString, launchContext = this@MainActivity) { ok, err -> }
```

## 7. Qualification API

```kotlin
pf.qualify { result ->
    result.onSuccess { resp ->
        Log.d("PF", "qualified tools: ${resp.qualifiedTools}")
    }.onFailure { Log.e("PF", "qualify failed", it) }
}

pf.getQualifiedTools { result ->
    result.onSuccess { ids -> /* ... */ }
}
```

## 8. Feedback widget backend

```kotlin
pf.submitFeedback(
    message = "Love the new onboarding",
    email = "user@example.com",
    metadata = mapOf("screen" to "home")
) { result ->
    result.onSuccess { /* submitted */ }
}
```

## Related

- [getting-started.md](getting-started.md) — full setup checklist  
- [api-reference.md](api-reference.md) — method reference  
