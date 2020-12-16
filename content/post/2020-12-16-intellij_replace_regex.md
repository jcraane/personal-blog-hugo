---
layout:     post

title:      "IntelliJ find and replace using regular expressions"
subtitle:   "IntelliJ: Examples of replacing text using Intellij find/replace with regular expressions"
author: Jamie Craane
date:       2020-12-16
description: "This post provides some example of using regular expressions with IntelliJ find and replace function."
#image: "https://img.zhaohuabing.com/in-post/2018-06-02-istio08/background.jpg"
published: true
showtoc: false
tags:
- IntelliJ

categories: [ IntelliJ ]
URL: "/2020/12/16/intellij_replace_regex/" 
---
IntelliJ (and all JetBrains ideas) support the ability to find and replace text using regular expressions. Regular expressions in this context can be very powerful. When you do not use regular expressions enough, it may take some time to come up with a correct regex, often looking at the [docs](https://www.jetbrains.com/help/idea/tutorial-finding-and-replacing-text-using-regular-expressions.html) or examples to find out how they work.

find/replace regex (and also regular find/replace) works particularly well when lots of replacements need to be made. 

This post gives some examples to show the possibilities using regex with find and replace. Use these as-is or as inspiration to create your own. Every example comes with a short description of its usage.
 
### Turning a date string into an object

Consider the following code:

```java
val date = "2020-12-16"
```

and we want to replace this with the following code:


```java
val date = LocalDate(2020, 12, 16)
```

We can use the following find/replace regex. The find regex finds the pattern 2020-10-10. The year is matched using (\d{4}) which matches exactly four digits and groups them because the \d{4} is put between parenthesis. We can use this group in the replace field. Note that $1 references the first group. $0 matches the entire regular expression.

The replacement pattern replaces it with LocalDate and uses the capturing groups from the find regex to fill in the year, month and day.

```java
Find: \"(\d{4})-(\d{2})-(\d{2})\"
Replace: LocalDate($1, $2, $3)
```

Please note that the above regex makes no distinction between 0 and 02. 02 needs to be replaced or else LocalDate(2020, 02, 12) will fail to compile.

**Resources**
- https://www.jetbrains.com/help/idea/tutorial-finding-and-replacing-text-using-regular-expressions.html
- [Structural search and replace](/2014/02/08/intellij_structural/)
