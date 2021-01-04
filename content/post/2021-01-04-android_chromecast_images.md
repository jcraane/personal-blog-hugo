---
layout:     post

title:      "Casting images with Chromecast"
subtitle:   "Android: example of how to cast images using the Chromecast SDK"
author: Jamie Craane
date: 2021-01-04       
description: "This post provides some example of using regular expressions with IntelliJ find and replace function."
#image: "https://img.zhaohuabing.com/in-post/2018-06-02-istio08/background.jpg"
published: true
showtoc: false
tags:
- Android
- Chromecast

categories: [ Android ]
URL: "/2021/01/04/android_chromecast_images/"
---

By adding Chromecast support to media rich apps like photo- and video players, users can display this content to Chromecast enabled devices. This greatly enhances the user experience of those apps.

This post demonstrates how to integrate the Chromecast SDK to add cast support for images. The app displays a list of publicly available images using an image carousel. When the user scrolls through those images, the images are cast to a connected device.

## Prerequisites, setup cast SDK

Before the app can cast images to a Chromecast device, the cast developer SDK must be setup. This includes a one-time fee of 5$. 

Execute the following steps to register the application:

1. When the developer free is paid, navigate to the [Google Cast SDK Developer Console](https://cast.google.com/publish/#/overview) and register your application. The only required field is the application name, as in the following example:
   ![img.png](/img/posts/setup-cast-application.png)
2. To test the application without plublishing it, register a Cast Receiver Device. Both receiver and app need to be on the same network.
3. Add the applicationId from step 1 to the local.properties file:

```plain
cast_app_id=<YOUR_GOOGLE_CAST_SDK_APP_ID>
```

## Implementing the application

**Adding support for Chromecast SDK**

The first step is to include the correct dependencies for the cast SDK:

```kotlin
implementation("androidx.mediarouter:mediarouter:1.0.0")
implementation("com.google.android.gms:play-services-cast-framework:17.0.0")
```

The sample app uses all default cast functionality. This section provides a brief overview of the steps to add cast support to an Android app.

The first step is to register an implementation of the CastOptions interface to the manifest. This initializes the CastContext singleton. The cast application id is required when initializing the cast context. Below is the implementation of the CastOptions interface in the sample app:

```java
public class CastOptionsProvider implements OptionsProvider {

    public CastOptions getCastOptions(Context context) {
        return new CastOptions.Builder()
                .setReceiverApplicationId(context.getString(R.string.cast_app_id))
                .setCastMediaOptions(new CastMediaOptions.Builder().build())
                .build();
    }

    @Override
    public List<SessionProvider> getAdditionalSessionProviders(Context context) {
        return null;
    }

}
```

The following line registers this implementation in the manifest:

```xml
<meta-data
   android:name="com.google.android.gms.cast.framework.OPTIONS_PROVIDER_CLASS_NAME"
   android:value="nl.jcraane.simpleimagecast.cast.CastOptionsProvider" />
```

# Resources
- [Google Cast SDK Developer Console](https://cast.google.com/publish/#/overview)
- [Cast-enable an Android app](https://codelabs.developers.google.com/codelabs/cast-videos-android/#0)
