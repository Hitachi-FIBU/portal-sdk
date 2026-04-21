# Hitachi Portal Integration (Android)

**Doc Version:** 0.7 · **SDK Version:** 1.0.30+ · **Confidentiality:** For Internal Use Only

> This document contains proprietary information that is confidential to Hitachi Asia. Disclosure of this document in full or in part may result in material damage to Hitachi Asia. Written permission must be obtained from Hitachi Asia prior to the disclosure of this document to a third party.

## Document Change Control

| Version | Date | Author(s) | Description |
|---------|------|-----------|-------------|
| 0.1 | 15 July 2024 | Chris Lee | Initial Version |
| 0.2 | 30 July 2024 | Chris Lee | Update addEventListener |
| 0.3 | 6 Aug 2024 | Chris Lee | Update Notify Listener |
| 0.4 | 17 Oct 2024 | Chris Lee | Update Permission Required and Additional Configuration for Portal Open |
| 0.5 | 22 Oct 2024 | Chris Lee | onRequestPermissionsResult and onActivityResult |
| 0.6 | 5 Mar 2026 | Chris Lee | Add Geolocation support |
| 0.7 | 20 Apr 2026 | Chris Lee | Add close-event `activityPerformed` callback payload |

## 1. Introduction

### 1.1 Overview

This document provides a comprehensive overview of the HASPortal SDK, highlighting its primary features and benefits for Android development. The SDK includes functionalities such as Wrapping, Token Interceptor, and Publish-Subscribe to enhance the overall performance, security, and scalability of the application.

**Objectives:**

- **Encapsulation and Security:** The SDK ensures that the portal functionality is encapsulated within a secure environment, providing better control over lifecycle events and consistent integration across different parts of the application.
- **Enhanced Security and Streamlined Authentication:** The Token Interceptor automates the management of authentication tokens, simplifying user session handling and ensuring secure communication between the app and backend services.
- **Asynchronous Communication and Decoupling:** The Pub-Sub mechanism facilitates non-blocking, event-driven communication, promoting a modular and scalable architecture by decoupling event producers and consumers.

**Key Benefits:**

- **Encapsulation:** Keeps portal interactions secure and consistent, with enhanced control over initialization, usage, and termination.
- **Security and Efficiency:** Token management, reducing the risk of errors and ensuring secure network communications.
- **Modularity and Flexibility:** Supports dynamic addition and removal of event listeners, improving application responsiveness and maintainability through asynchronous event handling.

### 1.2 What's New in v0.7

1. **Close-event `activityPerformed` callback payload**
   - The `eventsCallbacks` listener now delivers an additional `activityPerformed: List<String>` value on the `name: "close"` callback, naming the server-confirmed activities the user completed before closing (e.g. `["points-redeemed"]`).
   - Host apps use this to decide what to refresh or log after the portal dismisses.
   - See [Section 4.3](#43-close-event-payload-activityperformed) for the listener sample.

## 2. Specification

- **Android minimum SDK:** 24
- **Minimum HASPortal SDK version:** 1.0.30 (for the features described in this document — see §1.2)
- **SDK Size:** 27kb

## 3. Installation

### 3.1 Add the AAR Module to Your Project

- Open your project in Android Studio.
- Place the AAR file in the `app/libs` directory of your project.
  - Create the `libs` directory inside `app` if it does not exist.

Project structure:
```
android [My Application]
├── .gradle
├── .idea
├── app
│   ├── build
│   ├── libs
│   │   └── HASPortal-release.aar
│   ├── src
│   ├── .gitignore
│   ├── build.gradle.kts
│   └── proguard-rules.pro
└── build
```

### 3.2 Update Build Settings

- Open the **build.gradle** / **build.gradle.kts** file of your app.
- Add the following lines to the dependencies section to include the AAR module:

For build.gradle:
```groovy
dependencies {
    implementation("androidx.appcompat:appcompat:1.4.1")
    implementation("androidx.browser:browser:1.4.0")
    implementation("com.google.android.material:material:1.5.0")

    implementation files("libs/HASPortal-release.aar")
}
```

For build.gradle.kts:
```kotlin
dependencies {
    implementation("androidx.appcompat:appcompat:1.4.1")
    implementation("androidx.browser:browser:1.4.0")
    implementation("com.google.android.material:material:1.5.0")

    implementation(files("libs/HASPortal-release.aar"))
}
```

### 3.3 Permission Required

- Add the following permissions for SDK usage
- Skip if the permissions are added

AndroidManifest Requirements:
```xml
<uses-feature android:name="android.hardware.camera" android:required="false" />
<uses-permission android:name="android.permission.CAMERA"/>
<uses-permission android:name="android.permission.INTERNET"/>

<!-- Required for geolocation -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<!-- Optional: allows approximate location as a fallback -->
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
```

**Note:** `ACCESS_FINE_LOCATION` is **required** for the SDK's geolocation feature to work. Without it, the runtime permission request will fail silently and the WebView will not receive location data. `ACCESS_COARSE_LOCATION` is optional and provides approximate location as a fallback.

Add `onRequestPermissionsResult` and `onActivityResult` for permission result handling for ComponentActivity:

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
    }

    override fun onRequestPermissionsResult(
        requestCode: Int,
        permissions: Array<out String>,
        grantResults: IntArray
    ) {
        portal.onRequestPermissionsResult(requestCode, permissions, grantResults)
    }

    override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode, resultCode, data)

        // Forward to Portal if it exists
        if (::portal.isInitialized) {
            portal.handleActivityResult(requestCode, resultCode, data)
        }
    }
}
```

## 4. Example Usage

### 4.1 Basic Portal

Below is an example of how to use the integrated SDK in your Android project:

```kotlin
package com.has.myapplication

import android.content.Context
import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.Button
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.platform.LocalContext
import com.has.portal.Portal

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            Surface(color = MaterialTheme.colorScheme.background) {
                PortalButton()
            }
        }
    }
}

@Composable
fun PortalButton() {
    val context = LocalContext.current
    val activity = context as? ComponentActivity

    Button(
        onClick = { activity?.let { openPortal(context, it, "https://example.com") } },
        modifier = Modifier.padding(8.dp)
    ) {
        Text("Loyalty Program")
    }
}

fun openPortal(context: Context, activity: ComponentActivity, url: String) {
    val configuration: MutableMap<String, Any> = mutableMapOf(
        "accessToken" to "access token",
    )

    val portal = Portal(context, activity, configuration)

    portal.addEventListener("eventsCallbacks") { data ->
        println("Listener event data: $data")

        val name = data["name"] as? String
        val type = data["type"] as? String

        when {
            name == "redeem" && type == "navigation" -> {
                println("Handling redeem navigation event")
                portal.close()
                // Handle the redeem navigation event here
            }
            name == "close" -> {
                println("Handling close event")

                // Read activityPerformed payload (see §4.3 for the convention).
                // Arrives as a JSONArray because WebAppInterface.jsonToMap
                // preserves JSON array values verbatim.
                val activityPerformed = when (val raw = data["activityPerformed"]) {
                    is org.json.JSONArray -> List(raw.length()) { raw.optString(it) }
                    is List<*> -> raw.filterIsInstance<String>()
                    else -> emptyList()
                }

                if (activityPerformed.isNotEmpty()) {
                    println("Activity performed: $activityPerformed")
                    // e.g. telemetry, refresh loyalty points, tier info, etc.
                } else {
                    println("No activity — user browsed and closed")
                }
            }
            name == "token-expired" -> {
                println("Handling Invalid Token event")
                // Handle the access token expired event here
                val newConfiguration: MutableMap<String, Any> = mutableMapOf(
                    "accessToken" to "new token",
                )
                portal.notifyListener("token-updated", newConfiguration)
            }
        }
        null
    }
    portal.open(url)
}
```

### 4.2 Geolocation

The SDK supports `navigator.geolocation` in web content loaded within the portal. The SDK handles the WebView geolocation delegate automatically — no native event code is required from the host app.

#### Host App Setup

Add location permissions to your `AndroidManifest.xml` (see [Section 3.3](#33-permission-required)).

The `onRequestPermissionsResult` override (already required for camera) automatically forwards location permission results to the SDK. No additional native code is needed.

#### Web-side Code (JavaScript)

Use the standard browser Geolocation API — no custom events required:

```javascript
// Get current position
navigator.geolocation.getCurrentPosition(
  function(position) {
    console.log('Latitude:', position.coords.latitude);
    console.log('Longitude:', position.coords.longitude);
    console.log('Accuracy:', position.coords.accuracy, 'meters');
  },
  function(error) {
    console.error('Geolocation error:', error.message);
  },
  {
    enableHighAccuracy: true,
    timeout: 10000,
    maximumAge: 0
  }
);

// Watch position (continuous updates)
var watchId = navigator.geolocation.watchPosition(
  function(position) {
    console.log('Updated position:', position.coords.latitude, position.coords.longitude);
  },
  function(error) {
    console.error('Watch error:', error.message);
  }
);

// Stop watching
navigator.geolocation.clearWatch(watchId);
```

**Permission Flow:**
1. Web content calls `navigator.geolocation.getCurrentPosition()` or `watchPosition()`
2. The SDK's `onGeolocationPermissionsShowPrompt` is triggered
3. If permission already granted: location data flows directly to the web callback
4. If not granted: the SDK requests `ACCESS_FINE_LOCATION` at runtime via a headless fragment
5. If denied: the SDK shows a Settings redirect dialog
6. Permission result is forwarded back to the WebView geolocation callback

**Important Notes:**
- `ACCESS_FINE_LOCATION` **must** be declared in `AndroidManifest.xml`, otherwise the runtime permission request will fail silently
- Unlike contacts and other custom permissions, geolocation does **not** use the custom event system (`cap-events`). It works through the standard `navigator.geolocation` browser API
- The SDK only requests `ACCESS_FINE_LOCATION` at runtime. If your app only declares `ACCESS_COARSE_LOCATION`, the SDK's permission check will not match and geolocation will be denied

### 4.3 Close Event Payload (`activityPerformed`)

The `eventsCallbacks` listener now receives an additional `activityPerformed` key on the `name: "close"` callback. It's a `List<String>` (delivered as a `JSONArray`) naming server-confirmed activities the user completed before the portal closed — e.g. `["points-redeemed"]` or `["points-redeemed", "tier-upgraded"]`. Empty or absent means no activity worth refreshing on.

The host app uses this to decide what to refresh or log once the portal dismisses — e.g. refetch loyalty points after a redemption, update tier after a qualifying action. Valid values are agreed with the web team; unknown entries should be ignored or trigger a conservative blanket refresh.

#### Reading the payload

```kotlin
portal.addEventListener("eventsCallbacks") { data ->
    val name = data["name"] as? String
    if (name == "close") {
        val activityPerformed = data["activityPerformed"] as? org.json.JSONArray
        // Log / telemetry / trigger refresh
        Log.d("HostApp", "Portal closed with activity: $activityPerformed")
    }
    null
}
```

#### Native listener behaviour

The `eventsCallbacks` listener with `name: "close"` fires only when the portal is closed programmatically from in-page content. Native-initiated dismissals instead fire `closeEvent` on the Portal with `{ url }` only — see [Section 5.4](#54-addeventlistener).

| Portal close path | `eventsCallbacks` (`name: "close"`) fires? |
|-------------------|:-:|
| Close dispatched from in-page content (carries `activityPerformed`) | ✅ |
| Hardware back button | ❌ |
| Navigation close button | ❌ |
| Close modal (confirmed) | ❌ |

## 5. API

### 5.1 Portal

```java
public constructor(context: Context, activity: Activity, configuration: MutableMap<String, Any>)
```

**Parameters**

| Param | Type | Additional Information |
|-------|------|----------------------|
| context | Context | The context of the application. |
| activity | ComponentActivity | The activity from which the portal is launched. |
| configuration | MutableMap&lt;String, Any&gt; | A mutable map containing configuration options such as `accessToken`, `title` and `showHeader`. |

**Sample**

```kotlin
val configuration: MutableMap<String, Any> = mutableMapOf(
    "accessToken" to "access token",
    "title" to "Portal", // optional
    "showHeader" to "true", // optional
)

val portal = Portal(context, activity, configuration)
```

### 5.2 open

```java
public void open(String urlString)
```

**Parameters**

| Param | Type | Additional Information |
|-------|------|----------------------|
| urlString | String | |

**Sample**

```kotlin
var urlString = "https://example.com"
portal.open(url)
```

### 5.3 close

```java
public void close()
```

**Sample**

```kotlin
portal.close()
```

### 5.4 addEventListener

```java
public void addEventListener(String eventName, Function<Map<String, Object>, Map<String, Object>> listener)
```

**Parameters**

| Param | Type | Additional Information |
|-------|------|----------------------|
| eventName | String | |
| listener | Consumer&lt;Map&lt;String, Object&gt;&gt; | |

**Sample**

```kotlin
portal.addEventListener("eventsCallbacks") { data ->
    println("Listener event data: $data")

    val name = data["name"] as? String
    val type = data["type"] as? String

    when {
        name == "redeem" && type == "navigation" -> {
            println("Handling redeem navigation event")
            portal.close()
            // Handle the redeem navigation event here
        }
        name == "close" -> {
            println("Handling close event")
            // Handle the close event here
        }
        name == "token-expired" -> {
            println("Handling Invalid Token event")
            // Handle the access token expired event here
            val newConfiguration: MutableMap<String, Any> = mutableMapOf(
                "accessToken" to "new token",
            )
            portal.notifyListener("token-updated", newConfiguration)
        }
    }
    null
}
```

### 5.5 notifyListener

```java
private void notifyListener(String eventName, Map<String, Object> data)
```

**Parameters**

| Param | Type | Additional Information |
|-------|------|----------------------|
| eventName | String | |
| data | Map&lt;String, Object&gt; | |

**Sample**

```kotlin
val newConfiguration: MutableMap<String, Any> = mutableMapOf(
    "accessToken" to "new token",
)
portal.notifyListener("token-updated", newConfiguration)
```

## Appendix: Migration from v0.6 to v0.7

If you're upgrading from version 0.6, here are the key changes:

### New Features
1. **`activityPerformed` callback payload** — the `eventsCallbacks` listener now receives an `activityPerformed: List<String>` value on the `name: "close"` callback, naming server-confirmed activities completed before close (e.g. `["points-redeemed"]`). See [Section 4.3](#43-close-event-payload-activityperformed).

### AndroidManifest Requirements
**None** — no new permissions required.

### Breaking Changes
**None** — v0.7 is fully backward compatible with v0.6. The new payload key is additive; existing listeners that do not read `activityPerformed` continue to work unchanged.

### Recommended Updates
1. Add a `name == "close"` branch to the existing `eventsCallbacks` listener and read `data["activityPerformed"]` if you want to react to specific activity (see [Section 4.3](#43-close-event-payload-activityperformed)).
