---
layout:     post

title:      "KMM adoption strategies"
subtitle:   "Several strategies exist when adopting Kotlin Multiplatform Mobile"
author: Jamie Craane
date: 2021-07-27       
description: "This post describes various strategies that can be used whena adopting KMM for native Android and iOS development."
image: "/img/beach.jpg"
published: true
showtoc: false
tags:
- Android
- iOS
- KMM

categories: [ KMM ]
URL: "/2021/07/27/kmm_adoption_strategies/"
---
# KMM adoption strategies

2021-11-11 08:40

## Introduction
[Kotlin multiplatform Mobile](https://kotlinlang.org/docs/kmm-getting-started.html#start-kmm-from-scratch), in short KMM, lets you write native cross platform mobile applications. It lets you share logic across iOS and Android while still leveraging all of the native features of the platform.

There are several advantages of sharing code between platforms:

- Reduced development time
- More consistent logic across platforms
- More consistent behavior across platforms
- Less bugs
- Improved team communication

KMM gives the flexibility to decide what code is shared. Without going into the details about what to share, some common strategies are:

1. business logic
2. networking and serialization
3. logging and error handling
4. localized resources and images with [kmm-resources](https://github.com/jcraane/kmm-resources) and [kmm-images](https://github.com/jcraane/kmm-images)
5. viewmodels

These code sharing strategies are divided into 5 levels, the higher the level, the more code is shared.

## Different adoption strategies
There are several strategies you can choose from when adopting KMM. A couple of the most common adoption strategies are described in this article. Some of the questions you can ask when choosing an adoption strategy are:

- Is there an existing Android or iOS app which you want to port to the other platform?
- Are there existing Android and iOS apps for which you would want to leverage code re-use?
- How can I develop native apps and still benefit from sharing code between those platforms?
- I have existing business logic written in Kotlin, how can I Leverage that in my native iOS and Android apps?
- How can I enable a faster time to market with native app development?
- How to create more consistency between iOS and Android?

### 1 Integrate KMM in an existing app
Adopting KMM does not mean rewriting the Android and iOS apps from scratch. It can be integrated gradually into an existing codebase at different levels. Most of the time, starting with the business logic (level 1) makes the most sense since this type of code is usually independent of platform **specific libraries**.

Depending on the networking and serialization library used, the next step is moving the networkingand serialization code to the shared code (level 2). If for example [ktor](https://ktor.io/) and [Kotlin serialization](https://github.com/Kotlin/kotlinx.serialization) are used, this should be straightforward because they both are multiplatform libraries.

If on the other the networking code does not used a multiplatform library yet, the networking code can be migrated in phases. Start by migrating some trivial use case, for example a simple GET request to retrieve some data and move that to the shared code.

After some code is moved or migrated to the shared library, it is a good upportunity to integrate it in the apps. For Android this is straightforward because the shared library is just exposed as an Android library within the IDE. It is just Kotlin code after all.

For iOS this is done by [integrating](https://kotlinlang.org/docs/kmm-integrate-in-existing-app.html) the library into the iOS project. After the integration is complete, the shared code can be imported and used in Swift or Objective C code as if it was a regular iOS framework.

I have used the above strategy succesfully to migrate code from an existing Android app to a shared library. iOS was then implemented using the code from this shared library which reduced the iOS development time significantly.

### 2 Start with Android and iOS at the same time
When creating a new app, KMM can be used right from the start. With this strategy both iOS and Android apps are created in parallel. A selection of multiplatform libraries should be made depending on the type of code to be shared across platforms.

Some examples of multiplatform libraries are:
- https://ktor.io/ For creation of multiplatform asynchronous client and server applications.
- https://github.com/Kotlin/kotlinx-datetime A multiplatform Kotlin library for working with date and time.
- https://github.com/Kotlin/kotlinx.serialization Multiplafform serialization library.
- https://github.com/cashapp/sqldelight Typesafe, multiplatform SQL library.

The advantage of this approach is that the shared code is implemented from the start. This enables Android and iOS to both have influence on the structure of the shared code. This, in my opinion, also enables tight integration between the whole team (iOS and Android) which ultimately improves communication and quality.

A variation of this approach is that further in the project development timeline, usually multiple features are developed at once. At this stage a single feature can be developed within the shared code and for a given platform of choice (Android or iOS). When development has made sufficient progress, the other platform can connect to the shared code, drastically reducing development time.

When working with features in parallel, use a divide and conquer strategy to distribute the features to be developed across both platform. For some features iOS will be started first, for other features Android will be first.

We also used this strategy successfully. The iOS and Android app of Eneco (Dutch Energy Company) is implemented from scratch using KMM.

### 3 Start with a specific platform first
With this third approach, development is started with a specific platform first. This strategy has quite some resemblance with the second one. The difference is that development of the second platform is postponed instead of development taking place immediately in parallel.

This has both pros and cons. The advantage is that the learnings from the development (or part of) of one platform can be applied directly to the development of the other platform. When the development of the second platform starts, all of the learnings so far can be applied directly without re-inventing the wheel. Probably most learnings will come from the UI part of the system, re-interating based on learnings and feedback from users (in whatever form that may be).

The disadvantage might be the slight delay in the start of the development of the second platform. Delaying development of the second platform does not neccesarily mean postponing development till the first platform is feature complete. Development of the second platform can start for example when the first (or couple) of features are implemented.

Several factors might influence the choice of development platform for which is developed first. Think of an existing user base for example from the website. Experience of the team or a general preference towards one specific platform. No hard rules exist here.

## Conclusion
Development with Kotlin Multiplatform Mobile has several advantages compared to develop two (Android and iOS) apps in isolation. With KMM code is shared across platforms which means faster time to market, less bugs, more consistent apps and a faster time to market.

There are several strategies to start with and adopt KMM. KMM can be integrated in existing apps or used in the development of completely new apps.

The takeway is that adopting KMM can be done at every stage in development and is mostly not intrusive. This means that complete or huge rewrites before KMM can be adopted are most of the time not needed. Because of this there are almost always tangible benefits when adopting KMM.

## Glossary
- **KMM** is an abbreviation for Koltin Multiplatform Mobile.
- **Platform specific library** Library which does not support multi-platform and thus only works on Android or iOS, but not both.
- **Multiplatform enabled library** Library specifically designed for KMM which works for both Android and iOS (and potentially tje JVM, web and native platforms).

## References
1. https://kotlinlang.org/docs/kmm-introduce-your-team.html
