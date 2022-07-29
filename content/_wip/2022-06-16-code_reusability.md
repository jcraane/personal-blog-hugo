---
layout:     post

title:      "Code re-usability"
subtitle:   "Code re-usability enabled faster time to market when done right"
author: Jamie Craane
date: 2021-11-15    
description: "This post describes various strategies on how to approach reusability in software products."
image: "/img/beach.jpg"
published: true
showtoc: false
tags:
- Android
- iOS
- KMM

categories: [ PROCESS ]
URL: "/2022/06/16/code_reusability/"
---

Re-usability within software development means to reuse (part of) a solution in different products with the aim of:
1. faster time to market
2. lower costs
3. less chance of introducing issues
4. be able to use shared knowledge

## Re-usability at different abstraction levels

Re-usability exists at many levels of abstraction in software development. Each abstraction offers some degree of advantages as well as disadvantages. Some examples, from high to low abstraction level:

**Infrastructure**
Using infrastructure to distribute software. This can be a cloud solution, but also an app store, for example.

**Reuse at API level**
APIs are the interface of an organization or service to the outside world. A good API abstracts the underlying business logic so that this underlying service can be easily integrated into existing services (apps, websites, btb, etc.). This is usually where the most opportunities in the organization lie. Too often there is too much business logic in consuming systems which results in: 

- longer time to marker
- too complex logic in client systems
- duplicated logic and therefore a chance of inconsistencies and a higher number of issues
- more difficult to transfer
 
**Reusability at SDK level**. 
As an example within app development, one should think of the Android and Apple SDKs. By making use of this, a large part is extracted for the developer: think of the mechanism for building UIs, push notifications, location based services etc.
 
**Reusability on libraries and frameworks**. 
Libraries and frameworks can be open source or close source (or both). By using libraries, solutions for existing problems are re-used instead of development from scratch. When adopting/evaluating a library you should ask yourself a couple of question to find out of the library does indeed provide the required value. Some actions/questions might be:

- Does the library solve the problem at hand (is it actually useful/)
- Checkout source code (does it not do anything tricky that might be a risk/liability)
- Does it have automated tests? Are they useful?
- How many open severe issues are there? Any open breaking bugs, crashes, edge cases)?
- Issues vs. data last maintained and commit activity.
- Tooling complexity of the things that make the library work

Commercial third party solutions  may require their own process to determine whether the solution provides added value.

Examples of open source libraries: Retrofit, Kotlin serialization, Ktor, Acompanist, SqlDelight
Examples of third party solutions: Notificare, Mapbox

**Reusability across platforms**. 
Today, several solutions exist to achieve cross-platform mobile development re-usability. For example when native app development is chosen: Kotlin Multiplatform Mobile is an option. See https://jamiecraane.dev/2021/11/15/kmm_adoption_strategies/ for more information.

**Reusability based on recommendations**. 
This includes architecture patterns and the toolset provided by platform SDKs or third party solutions. Think of MVVM architecture (which is the preferred architecture at the moment). Whether the solution from the platform or a third-party library is the best depends on many factors and outside the scope of this article.

**Reusability in custom software**. 
Here we have arrived at the lowest level where the actual specific solution of the problem is implemented. Re-use at this level is possible but is not an end in itself. The time it takes to make reusable components usually does not outweigh the benefits one gets. Of course, this does not mean that reuse does not take place at this level at all.

**Reuse within a single codebase**: 
Reuse within a single codebase is on an as-needed basis. A simple rule usually suffices: apply reuse when a certain piece of code is used more than twice. When using logic for the third time, sufficient knowledge is available to make this logic reusable (this also prevents over-engineering, among other things).

**Reuse within different code bases within an organization**: 
Sometimes it happens that there are defined pieces of functionality that can be reused. Think of an authentication flow or a specific analytics solution that has been chosen. Furthermore, reuse at this level does not serve an end in itself, but can best be completed according to need.
