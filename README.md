# Playsout Android SDK Integration Guide

This guide is for developers integrating the Playsout game center into an Android host app. The examples use Kotlin and the release published to Maven Central.

> SDK version: `0.1.1`  
> Maven coordinates: `com.playsout:playsout-android-sdk:0.1.1`

## 1. Requirements

The host project must meet these requirements:

| Item | Requirement |
| --- | --- |
| Android `minSdk` | 24 or higher |
| Android `compileSdk` | 37 or higher |
| Java compatibility | 11 |
| Language | The Kotlin examples require Kotlin coroutines support |

The host app must provide the platform login result: the platform identifier, platform user ID, platform login token, and username. `platformToken` must come from the host platform's real login flow. A sample token is not suitable for production and does not prove that platform authentication succeeded.

## 2. Add the dependency

Make sure the project repositories include Google Maven and Maven Central. Example `settings.gradle` configuration using Groovy DSL:

```groovy
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}
```

Add the published dependency to the host app module's `build.gradle`:

```groovy
dependencies {
    implementation 'com.playsout:playsout-android-sdk:0.1.1'
}
```

For Kotlin DSL, add this to `app/build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.playsout:playsout-android-sdk:0.1.1")
}
```

You do not need to include the SDK project in `settings.gradle` or add it as a local project dependency. The SDK manifest declares the required `INTERNET` permission, which is merged into the host app during the build.

## 3. Initialize the SDK

Initialize the SDK once in `Application.onCreate()`. Initialization must run on the main thread. A second initialization attempt in the same process throws `ALREADY_INITIALIZED`.

```kotlin
class HostApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        PlaysoutSdk.init(
            context = applicationContext,
            config = PlaysoutConfig(
                locale = null, // Follow the device language
                enableLogging = BuildConfig.DEBUG,
            ),
        )
    }
}
```

Declare the Application class on the host app's `<application>` element in `AndroidManifest.xml`:

```xml
<application
    android:name=".HostApplication"
    ... >
    ...
</application>
```

If the host app already has an `Application` subclass, add the initialization code to it instead of creating another one.

## 4. Log in to the SDK after host login

The SDK does not provide a platform account login screen. Complete the host platform login first, then pass the **real platform login information** to the SDK. `login` is a suspend function and must be called from a coroutine.

```kotlin
class MainActivity : ComponentActivity() {
    private fun onHostLoginSucceeded(
        platformUserId: String,
        platformToken: String,
        username: String,
    ) {
        lifecycleScope.launch {
            try {
                val user = PlaysoutSdk.login(
                    platform = "eros", // Currently "eros" or "grab"
                    platformUserId = platformUserId,
                    platformToken = platformToken,
                    username = username,
                )
                // Login succeeded. user.id is the Playsout user ID.
            } catch (error: PlaysoutSdkException) {
                when (error.code) {
                    PlaysoutErrorCode.NETWORK_ERROR,
                    PlaysoutErrorCode.TIMEOUT,
                    -> showRetryMessage()

                    else -> showLoginError(error.code)
                }
            }
        }
    }
}
```

Parameter requirements:

| Parameter | Requirement |
| --- | --- |
| `platform` | `eros` or `grab`, case-insensitive |
| `platformUserId` | Non-empty, up to 40 characters |
| `platformToken` | Non-empty, up to 8192 characters; pass the real token returned by the host platform |
| `username` | 1 to 20 characters after trimming leading and trailing whitespace |

After a successful login, the SDK persists the Playsout session. The host app remains responsible for its own platform account state. Do not use `PlaysoutUser.id` in place of the platform user ID or platform token.

## 5. Open the game center

Call this from an Activity on the main thread:

```kotlin
openGameCenterButton.setOnClickListener {
    PlaysoutSdk.openGameCenter(this)
}
```

The SDK opens and manages the game center and game screens. **In the current version, users must be logged in before they can launch a game.** Users can open the game center and browse the list while logged out, but tapping a game displays a login prompt and does not open the game. Complete the SDK login before opening the game center if users should be able to play.

## 6. Manage login state

The SDK stores the Playsout session encrypted. After initialization, you can query the local session state:

```kotlin
if (PlaysoutSdk.isLoggedIn()) {
    val user = PlaysoutSdk.getUser() // Local snapshot; no network request
}
```

You can also log out:

```kotlin
PlaysoutSdk.logout()
```

If you observe session changes, remove the listener when the registering component is no longer active so it is not retained unnecessarily:

```kotlin
class MainActivity : ComponentActivity() {
    private val authListener = object : PlaysoutAuthStateListener {
        override fun onLogin(user: PlaysoutUser) {
            // SDK login succeeded
        }

        override fun onLogout() {
            // SDK session was cleared
        }

        override fun onAuthExpired(error: PlaysoutSdkException) {
            // Get the host platform identity again, then call PlaysoutSdk.login.
        }
    }

    override fun onStart() {
        super.onStart()
        PlaysoutSdk.addAuthStateListener(authListener)
    }

    override fun onStop() {
        PlaysoutSdk.removeAuthStateListener(authListener)
        super.onStop()
    }
}
```

Switch to the main thread before updating UI from a callback. The SDK does not repeat the host platform login for you. The host app is responsible for obtaining platform identity and logging in again after the session expires.

## 7. Set the language

`PlaysoutConfig.locale` can pin the initial language. When it is `null`, the SDK uses the user's saved choice or the system language. Supported codes:

| Code | Language |
| --- | --- |
| `en` | English |
| `zh` | Simplified Chinese |
| `ja` | Japanese |
| `ko` | Korean |
| `th` | Thai |
| `vi` | Vietnamese |
| `id` | Indonesian |
| `ms` | Malay |

Change the language at runtime on the main thread:

```kotlin
PlaysoutSdk.setLocale("zh")
```

Only the two-letter codes in the table are accepted. For example, `zh-CN` throws `INVALID_ARGUMENT`.

## 8. API reference

| API | Description | Thread/call requirement |
| --- | --- | --- |
| `init(context, config)` | Initialize the SDK | Main thread; once per process |
| `login(platform, platformUserId, platformToken, username)` | Log in using the host platform identity | Suspend function; call from a coroutine |
| `logout()` | Clear the SDK login session | Regular function |
| `isLoggedIn()` | Check local session state | Regular function |
| `getUser()` | Get the local user snapshot | Regular function; no network request |
| `getUserInfo()` | Fetch the latest user information | Suspend function; call from a coroutine |
| `openGameCenter(activity)` | Open the game center | Main thread |
| `setLocale(locale)` | Change the language | Main thread |
| `getLocale()` | Get the current language code | Regular function |
| `addAuthStateListener(listener)` | Register an authentication state listener | Main thread |
| `removeAuthStateListener(listener)` | Remove an authentication state listener | Main thread |

All APIs other than `init` throw `NOT_INITIALIZED` if called before SDK initialization.

## 9. Error handling

The SDK reports errors with `PlaysoutSdkException`. Use `error.code` in application logic; do not rely on the English `message` text.

| Error code | Meaning and suggested action |
| --- | --- |
| `NOT_INITIALIZED` | Call `init` first |
| `ALREADY_INITIALIZED` | Check for duplicate initialization |
| `INVALID_ARGUMENT` | Check argument format, length, and required calling thread |
| `AUTH_REQUIRED` | Complete host platform login and call SDK `login` |
| `AUTH_EXPIRED` | Get the host platform identity again and call SDK `login` |
| `NETWORK_ERROR` | Check the network and let the user retry |
| `TIMEOUT` | The request timed out; you may let the user retry |
| `API_ERROR` | The server rejected the request or returned an invalid response; check login parameters and service status |
| `SERVER_ERROR` | The service is temporarily unavailable; retry later |
| `UNKNOWN` | Unexpected error; record the error code and contact SDK support |

## 10. Troubleshooting

- **Tapping a game shows a login prompt:** Confirm `PlaysoutSdk.login` completed successfully. Logging into the host platform alone does not establish an SDK session.
- **`NOT_INITIALIZED`:** Confirm the Application initialization code ran; do not initialize only in a particular Activity screen.
- **`ALREADY_INITIALIZED`:** Centralize initialization in the Application instead of calling it each time an Activity is created.
- **Login returns `INVALID_ARGUMENT`:** Check the platform code, platform user ID, token, and username length, as well as the required calling thread.
- **Login fails:** Use real platform credentials, check the network, and handle the result using `error.code`. Do not use a demo token for production integration.
- **Images or games do not load:** Check that the device can reach the game resources and that the SDK's `INTERNET` permission was merged into the host app manifest.
- **Production logging:** Set `enableLogging` to `false`. Diagnostic logs are intended for development troubleshooting.
