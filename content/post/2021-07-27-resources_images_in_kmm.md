---
layout:     post

title:      "Sharing localized string resources and images between iOS and Android in Kotlin Multiplatform Mobile"
subtitle:   ""
author: Jamie Craane
date: 2021-07-27       
description: "This post show how to add share text resources and images between iOS and Android in KMP by utilizing the kmm-resources and kmm-images plugins."
image: "/img/cast-background.jpg"
published: false
showtoc: false
tags:
- Android
- Chromecast

categories: [ KMM ]
URL: "/2021/01/05/resource_images_kmm/"
---

## Introduction

With [Kotlin Multiplatform Mobile](https://kotlinlang.org/lp/mobile/) (in short KMM) allows you to share a lot of code which is normally duplicated across iOS and Android native platforms. KMP does mandate that certain type of code to be shared, it is up to you. It also does not mandate any particular architecture.

Although KMM enables sharing of code between platforms it does not provide the ability to share string resources and images out-of-the box between iOS and Android. This post describes one possible option to do exactly this.

This post describes how to achieve both sharing of localized resources and images between platforms using the Gradle plugins kmm-resources and kmm-images. Both plugins are written by [me](https://github.com/jcraane) and [Lammert Westerhoff](https://github.com/lammertw)`

## Sharing localized string resources between iOS and Android (and web) and shared code

One of the goals in sharing localized resources across platforms is that those resources can be used across platforms. Not only in those platforms itself (iOS and Android) but also within the code of the shared module. This creates the possibility to create view models in shared code which determine what localized resources to show in the view.

String resources (for example: dialog titles, button labels, toast messages etc) are by default not shared in a KMM project. The benefit of sharing those resources is consistent text labels across all platforms using those resources. This means less duplication, fewer bugs because of that duplication and a more consistent experience across platforms.

The challenge in using localized resources on different platforms is the different approaches each platform has regarding those localization resources. Android is using resource xml files, iOS uses asset catalogs, and the web can for example use property files. For this to work we created the kmm-resources Gradle plugin.

With the kmm-resources plugin you can define localized resources in the code of the shared module. At the moment yaml is used for this. The kmm-resources plugin then generates the code, so those resources can be used across all platforms, including the shared module. Below is a small example without going into too much detail (extensive documentation can be found on the project page of the kmm-resources plugin itself, including lots of examples.

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

## Conclusion



# Resources
- [KMM Resources](https://github.com/jcraane/kmm-resources)
- [KMM Images](https://github.com/jcraane/kmm-images)

