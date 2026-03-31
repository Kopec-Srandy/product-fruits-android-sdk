# Product Fruits Android SDK (Maven distribution)

Public **Maven repository layout** for `com.productfruits:sdk`, published automatically from the private Android SDK repo (same idea as [product-fruits-ios-sdk](https://github.com/product-fruits/product-fruits-ios-sdk) for SPM).

## Documentation

- **[Getting started](docs/getting-started.md)** — Gradle, config, identify, tracking, push  
- **[Examples](docs/examples.md)** — Application, Navigation, Compose, FCM permission  
- **[API reference](docs/api-reference.md)** — `ProductFruits`, `ProductFruitsConfig`, errors  

## Gradle (Kotlin DSL)

**Repository** (static files on `main`; use `raw.githubusercontent.com`):

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

**Dependency** (pick the version you need; tags/releases match `v1.x.x`):

```kotlin
dependencies {
    implementation("com.productfruits:sdk:1.0.5")
}
```

Use a **fixed version** (not `+`). Old versions stay under `maven-repo/com/productfruits/sdk/<version>/`.

## Firebase (push)

The SDK lists FCM as `compileOnly`. Your app should add:

```kotlin
implementation(platform("com.google.firebase:firebase-bom:<version>"))
implementation("com.google.firebase:firebase-messaging")
```

and register `com.productfruits.sdk.ProductFruitsFirebaseMessagingService` as in the SDK docs.

## Layout

- `maven-repo/` — standard Maven paths: `com/productfruits/sdk/<version>/sdk-<version>.aar`, `.pom`, etc.
- **Git tags** `v*` on this repo mark releases (and match GitHub Releases with an attached `.aar` when the workflow runs).

## Publishing (maintainers)

Releases are pushed by CI from the private SDK repository when you push a tag `v1.2.3` or run the workflow manually with a version string. Required secret: `MOBILE_DEPLOYMENT_GITHUB_ACCESS_TOKEN` (PAT with push access to this repo).
