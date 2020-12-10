---
layout:     post

title:      "IntelliJ structral search and replace"
subtitle:   "IntelliJ: the power of structural search and replace"
author: Jamie Craane
date:       2014-02-08
description: "Sometimes you run into a situation where you want to refactor some code bu"
#image: "https://img.zhaohuabing.com/in-post/2018-06-02-istio08/background.jpg"
published: true
showtoc: false
tags:
- IntelliJ

categories: [ IntelliJ ]
URL: "/2014/02/08/intellij_structural/"
---

Sometimes you run into a situation where you want to refactor some code but cannot use the regular refactorings. For example, take the following code:
```javascript
jQuery("body").on("change", "#fontSelector", "change", function () {
    var selectedFont = jQuery("#fontSelector").val();
    layoutDesigner.selectFont(selectedFont);
});
```
I wanted to replace the above code with the following:
```javascript
jQuery("#fontSelector").off("change");
jQuery("#fontSelector").on("change", function () {
    var selectedFont = jQuery("#fontSelector").val();
    layoutDesigner.selectFont(selectedFont);
});
```
Sure, I could rewrite this manually. But this takes a long time with dozens of such event handling constructs, all with different selectors, events and functions. Structural search and replace to the rescue.

But instead of writing how you could do this with structural search and replace, I recorded a little screencast which demonstrates the concept. The video can be found [here](https://www.youtube.com/watch?v=Jb-YNgDClKg)
