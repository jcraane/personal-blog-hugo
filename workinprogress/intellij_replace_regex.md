---
layout:     post

title:      "IntelliJ replace with regex"
subtitle:   "IntelliJ: Examples of replacing text using Intellij find/replace witg regex"
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
IntelliJ (and all JetBrains ideas) support the ability to find and replace text using regular expressions. Regular expressions in this context can be very powerful. When you do not use regular expressions enough, it may take some time to come up with a correct regex, often looking at the [docs](https://www.jetbrains.com/help/idea/tutorial-finding-and-replacing-text-using-regular-expressions.html) or example to find out how they work.

find/replace regex (and also regular find/replace) works particulary well when lots of replacements need to be made. 

This post gives some examples to show the possibilties using regex with find and replace. Use these as-is or as inspiration to create your own. Every example comes with a short description of its usage.
 
###Turning a date string into an object

Consider the following code:

```java
val date = "2020-12-16"
```

and we want to replace this with the following code:


```java
val date = LocalDate(2020, 12, 16)
```

We can use the following find/replace regex. The find regex finds the pattern 2020-10-10. The year is matched using (\d{4}) which matches exactly four digits and groups them because the \d{3} is between brackets. We can use this group in the replace field where $1 references this group ($0 matches the entire regular expression).

The replace plattern replaces it with LocalDate and uses the capturing groups from the find regex to fill in the year, month and day.

```java
Find: \"(\d{4})-(\d{2})-(\d{2})\"
Replace: LocalDate($1, $2, $3)
```

**Resources**
- https://www.jetbrains.com/help/idea/tutorial-finding-and-replacing-text-using-regular-expressions.html
