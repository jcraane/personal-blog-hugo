---
layout:     post

title:      "Kotlin: Multi variable null check"
subtitle:   "Different ways of checking for not null with multiple variables"
author: Jamie Craane
date:       2020-12-10
description: "There are several patterns you can use to test for not null on multiple variables. This post describes different methods of null checking multiple variables in Kotlin."
#image: "https://img.zhaohuabing.com/in-post/2018-06-02-istio08/background.jpg"
published: false
showtoc: false
tags:
- Kotlin

categories: [ Kotlin ]
URL: "/2020/12/10/kotlin_null/"
---
Although not-null, immutable variables with sensible defaults are often desired, it is not always the case. It is sometimes required to check if multiple variables are not null. This post describes several variants of how to do this when smart casts are not possible because the variables are mutable.

Consider the following example:
```kotlin
class Person() {
    var name: String? = null
    var age: Int? = null

    fun doSomething() {
        if (name != null && age != null) {
            name.length // Compile error since smart-cast is not possible
        }
    }
}
```
When the name or age property is accessed after the above null check we get the following compile error: mart cast to 'String' is impossible, because 'name' is a mutable property that could have been changed by this time.

There are several solutions to this problem which are presented below.

**Assign variables to immutable val**

In this variant the mutable variables are assigned to immutable val. This wat the compiler is able to execute the smart cast:
```kotlin
class Person {
    var name: String? = null
    var age: Int? = null

    fun doSomething() {
        val n = name
        val a = age
        if (a != null && n != null) {
            n.length // Smart cast is possible
        }
    }
}
```
**Use destructuring to assign the variables to a val**

In the example the destructuring capabilities of Kotlin are used to make assigment of multiple variables at once easier. The null-check itself is the same:
```kotlin
class Person {
    var name: String? = null
    var age: Int? = null

    fun doSomething() {
        val (n, a) = name to age
        if (a != null && n != null) {
            n.length
        }
    }
}
```
In the above example the to keyword is used to create a pair which is destructured into n and a. If there are more than two variables the listOf() method can be used to destructure up to 5 variables (because a list defines component1 to component5 functions) as in the following example:
```kotlin
val (a, b, c, d, e) = kotlin.collections.listOf("a", "b", "c", "d", "e")
```
**Use a specialized function**

The following example uses a function to check the passed-in variables for not-null and executes the block of both variables are not null:
```kotlin
/**
 * Executes block if all supplied params are not null.
 *
 * @param a First param
 * @param b Second param
 * @param block The block to execute of all of the supplied params are not null.
 */
fun <A, B, RESULT> doIfNotNull(a: A?, b: B?, block: (A, B) -> RESULT): RESULT? {
    if (a != null && b != null) {
        return block(a, b)
    }

    return null
}
```
The above function can be used like this:
```kotlin
class Person {
    var name: String? = null
    var age: Int? = null

    fun doSomething() {
        doIfNotNull(name, age) { n, a -> n.length}
    }
}
```

For a more in-depth discussion see https://discuss.kotlinlang.org/t/kotlin-null-check-for-multiple-nullable-vars/1946
