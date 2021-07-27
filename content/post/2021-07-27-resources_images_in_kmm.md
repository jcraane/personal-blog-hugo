---
layout:     post

title:      "Localized resources and images in KMM"
subtitle:   "Sharing localized string resources and images between iOS and Android in Kotlin Multiplatform Mobile"
author: Jamie Craane
date: 2021-07-27       
description: "This post show how to add share localized resources and images between iOS and Android in KMM project."
image: "/img/kmm-header.png"
published: false
showtoc: false
tags:
- Android
- iOS
- KMM

categories: [ KMM ]
URL: "/2021/07/27/resource_images_kmm/"
---

## Introduction

[Kotlin Multiplatform Mobile](https://kotlinlang.org/lp/mobile/) (in short KMM) allows you to share a lot of code which is normally duplicated across iOS and Android native platforms. KMM does not mandate that certain type of code is shared, it is up to you. It also does not mandate any particular architecture.

Although KMM enables sharing of code between platforms it does not provide the ability to share localized string resources and images out-of-the box between iOS and Android. 

This post describes how to achieve both sharing of localized resources and images between platforms using the Gradle plugins kmm-resources and kmm-images. Both plugins are written by [me](https://github.com/jcraane) and [Lammert Westerhoff](https://github.com/lammertw).

We decided to create two separate plugins to make it easy to use one or the other or both. This also allows us to keep the two implementations and life cycles separate which eases maintenance. We also did not decide to create a framework for this. Those plugins are not opinionated to a given architecture such as MVI or MVVM. They can be used in whatever architecture the apps use.  

## Sharing localized string resources between iOS and Android (and web) and shared code

One of the goals in sharing localized resources across platforms is that those resources can be used across platforms. Not only in those platforms itself (iOS and Android) but also within the code of the shared module. This creates the possibility to create view models in shared code which determine what localized resources to show in the view.

String resources (for example: dialog titles, button labels, toast messages etc) are by default not shared in a KMM project. The benefit of sharing those resources is: consistent text labels across all platforms using those resources. This means less duplication, fewer bugs because of that duplication and a more consistent experience across platforms.

The challenge in using localized resources on different platforms are the different approaches each platform has regarding those localized resources. Android is using resource xml files, iOS uses asset catalogs, and the web can for example use property files. For this to work we created the kmm-resources Gradle plugin.

With the kmm-resources plugin you can define localized resources in the code of the shared module. At the moment yaml is used for this. The kmm-resources plugin then generates the code, so those resources can be used across all platforms, including the shared module. Below is a small example without going into too much detail (extensive documentation can be found on the project page of the [KMM Resources](https://github.com/jcraane/kmm-resources) plugin itself, including lots of examples.

With the kmm-resources plugin configured in your shared module add a yaml file containing the resources:

```yaml
greetings:
  hello:
    nl: Hallo vanuit common code
    en: Hello from common code
```

Since the plugin is hooked into the pre-build task, the resources can be used immediately after the project is build:

```kotlin
// Use in shared code, when executing this code, the correct translation is returned based on the configued language
val message = L.greetings.hello()
```

## Sharing images between iOS and Android

Next to sharing localized resources we also wanted to share images across Android and iOS. for this we created the kmm-images plugin.

The goal of sharing images is to support a variety of images in the shared module which are converted to be used in Android and iOS. The images that are currently supported are:

- jpg
- png
- vector images (pdf)

Images are placed in a folder of choice in the shared module. jpg images are scaled according to the device capabilities. pdf images are converted to Android drawables on the Android platform (iOS uses pdf's as-is). At the moment the plugin relies on some external tools to do the image conversion. This might change in a future version.

A small example is given below:

Consider the following images in the shared module:

```text
shared
    images
        icon.pdf
        piano.jpg
```

After configuring the kmm-images plugin, which is hooked into the prebuild and compileKotlin* tasks, the images can be used in shared code:

```kotlin
class MainViewModel {
    fun createViewState() = MainViewState(
        title = "My Main Screen",
        image = Images.IC_FLAG_NL
    )
}

data class MainViewState(
    val title: String,
    val image: Image
)
```

The above example defines a view model in shared code which determines which image to show in the view (by referencing the generated image in the Images class). Extension functions exists in both platforms to actually render those images.

Extensive documentation and examples can be found at [KMM Images](https://github.com/jcraane/kmm-images) 

## Conclusion

This post shows how to use localized resources and images across both platforms and shared code in a KMM project. This is done using two separate Gradle plugins which use code generation to enable this.

# Resources
- [KMM Resources](https://github.com/jcraane/kmm-resources)
- [KMM Images](https://github.com/jcraane/kmm-images)

