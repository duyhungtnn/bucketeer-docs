---
title: Android reference
slug: /sdk/client-side/android
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

This category contains topics explaining how to configure Bucketeer's Android SDK.

:::info Compatibility

The Bucketeer SDK is compatible with Android SDK versions 21 and higher (Android 5.0, Lollipop).

:::

## Getting started

Before starting, ensure that you follow the [Getting started](/) guide.

### Implementing dependency

Implement the dependency in your Gradle file. Please refer to the [SDK releases page](https://github.com/bucketeer-io/android-client-sdk/releases) to find the latest version.

<Tabs>
<TabItem value="gradle" label="Gradle">

```groovy showLineNumbers
dependencies {
  implementation 'io.bucketeer:bucketeer-android-client-sdk:LATEST_VERSION'
}
```

</TabItem>
</Tabs>

:::info ProGuard

You may need to include the ProGuard configuration lines from [consumer-proguard-rules.pro](https://github.com/bucketeer-io/android-client-sdk/blob/master/bucketeer/proguard-rules.pro)
into your proguard file if the aar artifact doesn't include the configuration automatically.

:::

### Importing client

Import the Bucketeer client into your application code.

<Tabs>
<TabItem value="kt" label="Kotlin">

```kotlin showLineNumbers
import jp.bucketeer.sdk.android.*
```

</TabItem>
</Tabs>

### Configuring client

Configure the SDK config and user configuration.

<Tabs>
<TabItem value="kt" label="Kotlin">

```kotlin showLineNumbers
val config = BKTConfig.Builder()
  .apiKey("YOUR_API_KEY")
  .apiURL("YOUR_API_URL")
  .featureTag("YOUR_FEATURE_TAG")
  .build()

val user = BKTUser.Builder()
  .id("USER_ID")
  .build()
```

</TabItem>
</Tabs>

:::info Custom configuration

Depending on your use, you may want to change the optional configurations available in the **BKTConfig.Builder**.

- **pollingInterval** (Minimum 5 minutes. Default is 10 minutes)
- **backgroundPollingInterval** (Minimum 20 minutes. Default is 1 hour)
- **eventsFlushInterval** (Default is 30 seconds)
- **eventsMaxQueueSize** (Default is 50 events)

:::

:::note

The Bucketeer SDK doesn't save the user data. The Application must save and set it when initializing the client SDK.

:::

### Initializing client

Initialize the client by passing the configurations in the previous step.

<Tabs>
<TabItem value="kt" label="Kotlin">

```kotlin showLineNumbers
val client = BKTClient.initialize(this.application, config, user)
```

</TabItem>
</Tabs>

:::note

The initialize process starts polling the latest variations from Bucketeer in the background using the interval `pollingInterval` configuration. When your application moves to the background state, it will use the `backgroundPollingInterval` configuration.

:::

If you want to use the feature flag on Splash or Main views, and the user opens your application for the first time, it may not have enough time to fetch the variations from the Bucketeer server.

For this case, we recommend using the callback in the initialize method.

<Tabs>
<TabItem value="kt" label="Kotlin">

```kotlin showLineNumbers
val callback = object : BKTClientInterface.FetchEvaluationsCallback {
  override fun onSuccess() {
    val showNewFeature = client.booleanVariation("YOUR_FEATURE_FLAG_ID", false)
    if (showNewFeature) {
        // The Application code to show the new feature
    } else {
        // The code to run when the feature is off
    }
  }

  override fun onError(exception: BKTException) {
    // Handle the error
  }
}

// The callback will return without waiting until the fetching variation process finishes
val timeout = 1000 // Default is 5 seconds

val client = BKTClient.initialize(this.application, config, user, callback, timeout)
```

</TabItem>
</Tabs>

## Supported features

### Evaluating user

The variation method determines whether or not a feature flag is enabled for a specific user.<br />
To check which variation a specific user will receive, you can use the client like below.

<Tabs>
<TabItem value="kt" label="Kotlin">

```kotlin showLineNumbers
val showNewFeature = client.booleanVariation("YOUR_FEATURE_FLAG_ID", false)
if (showNewFeature) {
    // The Application code to show the new feature
} else {
    // The code to run when the feature is off
}
```

:::note

The variation method will return the default value if the feature flag is missing in the SDK.

:::

</TabItem>
</Tabs>

### Variation types

The Bucketeer SDK supports the following variation types.

<Tabs>
<TabItem value="kt" label="Kotlin">

```kotlin showLineNumbers
fun booleanVariation(featureId: String, defaultValue: Boolean): Boolean

fun stringVariation(featureId: String, defaultValue: String): String

fun intVariation(featureId: String, defaultValue: Int): Int

fun doubleVariation(featureId: String, defaultValue: Double): Double

fun jsonVariation(featureId: String, defaultValue: JSONObject): JSONObject
```

</TabItem>
</Tabs>

### Updating user variations

Sometimes depending on your use, you may need to ensure the variations in the SDK are up to date before evaluating a user.

The fetch method uses the following parameters.

- **Callback**
- **Timeout** (The callback will return without waiting until the fetching process finishes. The default is 5 seconds)

<Tabs>
<TabItem value="kt" label="Kotlin">

```kotlin showLineNumbers
val callback = object : BKTClientInterface.FetchEvaluationsCallback {
  override fun onSuccess() {
    val showNewFeature = client.booleanVariation("YOUR_FEATURE_FLAG_ID", false)
    if (showNewFeature) {
        // The Application code to show the new feature
    } else {
        // The code to run when the feature is off
    }
  }

  override fun onError(exception: BKTException) {
    // Handle the error
  }
}

// The callback will return without waiting until the fetching variation process finishes
val timeout = 1000 // Default is 5 seconds

client.fetchEvaluations(callback, timeout)
```

</TabItem>
</Tabs>

:::caution

Depending on the client network, it could take a few seconds until the SDK fetches the data from the server, so use this carefully.

You don't need to call this method manually in regular use because the SDK is polling the latest variations in the background.

:::

### Updating user variations in real-time

The Bucketeer SDK supports FCM ([Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging)).
Every time you change some feature flag, Bucketeer will send notifications using the FCM API to notify the client so that you can update the variations in real-time.

Assuming you already have the FCM implementation in your application.

<Tabs>
<TabItem value="kt" label="Kotlin">

```kotlin showLineNumbers
override fun onMessageReceived(remoteMessage: RemoteMessage?) {
  remoteMessage?.data?.also { data ->
    val isFeatureFlagUpdated = data["bucketeer_feature_flag_updated"]
    if (isFeatureFlagUpdated) {
      val callback = object : BKTClientInterface.FetchEvaluationsCallback {
        override fun onSuccess() {
          val showNewFeature = client.booleanVariation("YOUR_FEATURE_FLAG_ID", false)
          if (showNewFeature) {
              // The Application code to show the new feature
          } else {
              // The code to run when the feature is off
          }
        }

        override fun onError(exception: BKTException) {
          // Handle the error
        }
      }
      
      // The callback will return without waiting until the fetching variation process finishes
      val timeout = 1000 // Default is 5 seconds

      client.fetchEvaluations(callback, timeout)
    }
  }
}
```

:::note

1- You need to register your FCM API Key on the console UI. [See more](#).

2- This feature may not work if the user has the notification disabled.

:::

</TabItem>
</Tabs>

### Reporting custom events

This method lets you save user actions in your application as events. You can connect these events to metrics in the experiments console UI.

In addition, you can pass a double value to the goal event. These values will sum and show as <br />`Value total` on the experiments console UI. This is useful if you have a goal event for tracking how much a user spent on your application buying items, etc.

<Tabs>
<TabItem value="kt" label="Kotlin">

```kotlin showLineNumbers
client.track("YOUR_GOAL_ID", 10.50)
```

</TabItem>
</Tabs>

### Flushing events

This method will send all pending analytics events to the Bucketeer server as soon as possible. This process is asynchronous, so it returns before it is complete.

<Tabs>
<TabItem value="kt" label="Kotlin">

```kotlin showLineNumbers
client.flush()
```

</TabItem>
</Tabs>

:::note

In regular use, you don't need to call the flush method because the events are sent every 30 seconds in the background.

:::

### User attributes configuration

This feature will give you robust and granular control over what users can see on your application. You can add rules using these attributes on the console UI's feature flag's targeting tab. [See more](#).

<Tabs>
<TabItem value="kt" label="Kotlin">

```kotlin showLineNumbers
val attributes = mapOf(
  "app_version" to "1.0.0",
  "os_version" to "11.0.0",
  "device_model" to "pixel-5"
  "language" to "english",
  "genre" to "female",
)

val user = BKTUser.Builder()
  .id("USER_ID")
  .customAttributes(attributes)
  .build()

val client = BKTClient.initialize(this.application, config, user)
```

</TabItem>
</Tabs>

### Updating user attributes

This method will update all the current user attributes. This is useful in case the user attributes update dynamically on the application after initializing the SDK.

<Tabs>
<TabItem value="kt" label="Kotlin">

```kotlin showLineNumbers
val attributes = mapOf(
  "app_version" to "1.0.1",
  "os_version" to "11.0.0",
  "device_model" to "pixel-5"
  "language" to "english",
  "genre" to "female",
  "country" to "japan",
)

client.updateUserAttributes(attributes)
```

</TabItem>
</Tabs>

:::caution

This updating method will override the current data.

:::

### Getting user information

This method will return the current user configured in the SDK. This is useful when you want to check the current user id and attributes before updating them through [updateUserAttributes](#getting-user-information).

<Tabs>
<TabItem value="kt" label="Kotlin">

```kotlin showLineNumbers
val user = client.currentUser()
```

</TabItem>
</Tabs>

### Getting evaluation details

This method will return the evaluation details for a specific feature flag. This is useful if you need to know the variation reason or send this data elsewhere.

<Tabs>
<TabItem value="kt" label="Kotlin">

```kotlin showLineNumbers
val evaluationDetails = client.evaluationDetails("YOUR_FEATURE_FLAG_ID")
```

:::note

This method will return null if the feature flag is missing in the SDK.

:::

</TabItem>
</Tabs>