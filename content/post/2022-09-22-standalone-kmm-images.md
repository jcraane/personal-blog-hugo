---
layout:     post

title:      "Standalone KMM Images"
subtitle:   "Using KMM Images in a standalone Android project"
author: Jamie Craane
date: 2022-09-22
description: "This post describes the benefits of using KMM Images in a standalone Android project."
image: "/img/factory.JPG"
published: true
showtoc: false
tags:
- Android
- KMM

categories: [ KMM ]
URL: "/2022/07/27/standalone_kmm_images/"
---

Image by Dall-E: *"Factory which produces lots of the same products. Dark."*

## Context

https://jamiecraane.dev/2021/07/27/resource_images_kmm/ describes how to use the [kmm-images plugin](https://github.com/jcraane/kmm-images) to share images between a Kotlin Multiplatform Mobile (short KMM) Android and iOS app. You can also use kmm-images in a standalone Android app. This post describes the advantages of doing this.

## Benefits of using kmm-images in an Android app

Kmm-images unifies image handling in a KMM iOS and Android app. Using kmm-images in a standalone Android app has the following advantages:

- automatic conversion of SVG to Android vector drawables
- automatic conversion of PNG and JPG to all requirements densities
- support for PDF image format
- ready for Kotlin Multiplatform

### SVG conversion

To use an SVG in Android it must be converted to a vector drawable. To convert an SVG to a vector drawable you can:

- Click on New -> Vector Asset and select the SVG to convert.
- Use the resource manager to convert multiple SVG files at once.
- Use an online tool to convert SVG files to vector drawables.
- Use a command line tool to convert SVG files to vector drawables.

kmm-images automates this process. All SVG images present in the images folder manager by kmm-images are converted to vector drawables. After the conversion the images are placed in the drawable resource folder. kmm-images uses the Android Vector Drawable tool to do the conversion.

### PNG/JPG conversion

To create multiple version for different densities of a PNG or JPG you can:

- Ask the designer to supply the image in multiple densities.
- Export the image in multiple densities from the design tool of choice.
- Use a tool to convert the image to multiple densities.

kmm-support automatic creation of all required densities for a PNG or JPG. The only requirement is to provide the images in the xxxhdpi density. kmm-images convers the images for both Android and iOS.

### Support of PDF image format

To use a PDF image in Android you must convert it to a supported format. kmm-images does this conversion automatically. A PDF images is converted to an SVG. The SVG is then convert to a vector drawable for use in Android.

### Ready for Kotlin Multiplatform

When you integrate kmm-images into an Android project means no more refactoring when the Android app is migrated to Kotlin Multiplatform Mobile. 

## Integrating kmm-images in an Android app

To integrate kmm-images in an Android app add the following to the Android app build.gradle.kts file:

1. Add id("dev.jamiecraane.plugins.kmmimages") version "1.0.0-alpha11" to the plugins section.
2. Add the following section to configure kmm-images

```kotlin
kmmImagesConfig {
    imageFolder.set(project.projectDir.resolve("images"))
    sharedModuleFolder.set(project.projectDir)
    androidResFolder.set(project.projectDir.resolve("src/main/generated-res"))
    packageName.set("<YOUR_PACKAGE_NAME>")
    defaultLanguage.set("en")
    usePdf2SvgTool.set(true) // optional parameter
}
```
3. Hook the generateImages task to the preBuild task:

```kotlin
var generateImages = tasks["generateImages"]
tasks["preBuild"].dependsOn(generateImages)
```

## Known issues

Kmm-images generates code to retrieve images in a type-safe way. This is not supported yet for standalone Android apps. See https://github.com/jcraane/kmm-images/issues/20.

