---
layout:     post

title:      "MyBatis"
subtitle:   "Mapping a map"
author: Jamie Craane
date:       2013-09-20
description: "As some of you will know I am a huge fan of MyBatis. I have used it in a lot of projects and it never failed me. I like how you are in control of the SQL and the flexibility this brings by mapping result sets to classes instead of mapping tables to classes."
#image: "https://img.zhaohuabing.com/in-post/2018-06-02-istio08/background.jpg"
published: true
tags:
- Mybatis

categories: []
URL: "/2020/12/10/kotlin_null/"
---

As some of you will know I am a huge fan of MyBatis. I have used it in a lot of projects and it never failed me. I like how you are in control of the SQL and the flexibility this brings by mapping result sets to classes instead of mapping tables to classes.

Recently I wanted to map some columns from a table to actual typed properties of an object, and some columns to a property of type Map within that same object. Consider the following class:
```java
public class Person {
    private String firstName;
    private String lastName;

    private Map dynamicProperties;

    // Getters and setters omitted.
}
```
and the following query:
```sql
select firstname, lastname, pref_1, pref_2, pref_3 from person;
```
The following resultmap maps the result of the query to the Person class:
```xml
<resultMap id="personDynamicProperties" type="map">
    <result column="pref_1" property="pref_1"/>
    <result column="pref_2" property="pref_2"/>
    <result column="pref_3" property="pref_3"/>
</resultMap>

<resultMap id="personResult" type="Person">
    <result column="firstname" property="firstName"/>
    <result column="lastname" property="lastName"/>
    <association property="dynamicProperties" resultMap="personDynamicProperties"/>
</resultMap>
```

