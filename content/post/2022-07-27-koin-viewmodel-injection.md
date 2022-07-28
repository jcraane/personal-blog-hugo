---
layout:     post

title:      "Koin ViewModel Injection"
subtitle:   "Inject objects into Android ViewModel which depend on a CoroutineScope"
author: Jamie Craane
date: 2022-07-27    
description: "This post describes various methods to use dependent objects in an Android ViewModel which depend on the viewModelScope."
image: "/img/beach.jpg"
published: true
showtoc: false
tags:
- Android
- iOS
- KMM

categories: [ PROCESS ]
URL: "/2022/07/27/koin_viewmodel_injection/"
---

## Context

A ViewModel in Android (not to be confused with the term ViewModel in the MVVM architecture pattern) is a component scoped to the lifecycle
of another component, usually a fragment or an activity. To create maintainable code, the ViewModel delegates to other objects. When using
coroutines, those other objects might use suspending functions. Those functions are called from the ViewModel in a CoroutineScope. For a
ViewModel this usually is the viewModelScope. When the ViewModel is cleared, the viewModelScope is cancelled.

The above scenario works in the case all dependent objects can mark all async functions as suspended. This might not always be the case., Those dependent objects might also be used elsewhere or coming from a library. In this case it might make sense to pass a coroutine scope when initializing those objects. This post describes several methods (also ones which do not work as expected) to accomplish this.

### Application scenario

To test all scenario's I created a minimal application with the following scenario (the basis of the application is the navigator sample from Android Studio).

The app defines three tabs: home, dashboard and notifications. The app consists of a MainActivity and three fragments, one for each tab. The notifications tab contains a button which launches the MainActivity again (pressing back finishes this instance of the MainActivity). The view models are scoped to the fragments.

The button in the home fragment calls the RandomGenerator which generates a random number and set this in a MutableStateFlow. The flow is collected in the HomeFragment and displayed in a TextView.

The diagram visualizes this specific flow:

![img.jpg](/img/posts/koin-injection-app-flow.jpg)

The following class diagram visualizes the structure of the app:

![img.jpg](/img/posts/koin-injection-app-structure.jpg)

What do we want to achieve: 

Dependent components can be injected in an Android ViewModel so the dependent components can safely use the provided
viewModelScope without worrying about the lifecycle of this scope.

Println statements are added to appropriate points in the application to visualize what is happening.

## Use a lateinit var

Source code: https://github.com/jcraane/AndroidKoinInjectViewModelScope/tree/use_lateinit_var_for_viewmodelscope

Although I am not a strong proponent (although sometimes they have their uses) of a lateinit var, this might be the first thing to come to mind when passing a viewModelScope to a dependent object. The RandomGenerator looks like this:

```kotlin   
class RandomGenerator() {
    // This scope is initialized in the init of the HomeViewModel
    lateinit var scope: CoroutineScope

    private val _randomNumber = MutableStateFlow(Random.nextInt())
    val randomNumber: Flow<Int> = _randomNumber

    fun generate() {
        println("injection: RandomGenerate.generate ($scope) is active: ${scope.isActive}")
        scope.launch {
            delay(300.milliseconds)
            _randomNumber.value = Random.nextInt()
        }
    }
}
```

In this scenario the RandomGenerator is injected into the HomeViewModel and the scope is initialized in the init method of the HomeViewModel. See the below code:

```kotlin
class HomeViewModel : ViewModel(), KoinComponent {
    // Code omitted

    val randomGenerator: RandomGenerator by inject()

    init {
        println("injection: HomeViewModel.init set viewModelScope to $viewModelScope")
        randomGenerator.scope = viewModelScope
    }

    // Code omitted
}
```

For inject to work in a ViewModel we need to let the ViewModel implements the KoinComponent interface. What is further important to know is that the RandomGenerator is defined as a single in the Koin module.

If we run the application we see the following output:

```text
HomeViewModel.init set viewModelScope to androidx.lifecycle.CloseableCoroutineScope@67c7b3f <-- viewModelScope is set in init
HomeViewModel.generate randomGenerator instance is com.example.myapplication.ui.home.RandomGenerator@2586393
RandomGenerate.generate (androidx.lifecycle.CloseableCoroutineScope@67c7b3f) is active: true

HomeViewModel.init set viewModelScope to androidx.lifecycle.CloseableCoroutineScope@ed7012f <-- New activity is launched, new viewModelScope is set
HomeViewModel.generate randomGenerator instance is com.example.myapplication.ui.home.RandomGenerator@2586393 <-- still the same RandomGenerator instance
RandomGenerate.generate (androidx.lifecycle.CloseableCoroutineScope@ed7012f) is active: true
HomeViewModel.onCleared <-- Activity is destroyed and view model is cleared (scope @ed7012f is cancelled) 

HomeViewModel.generate randomGenerator instance is com.example.myapplication.ui.home.RandomGenerator@2586393 <-- still the same RandomGenerator instance
RandomGenerate.generate (androidx.lifecycle.CloseableCoroutineScope@ed7012f) is active: false <-- scope in RandomGenerator is not active
```

The last line shows that the scope in RandomGenerator is not active anymore. This is because when we started a new activity, the original HomeFragment was not destroyed (only onDestroyView was called). Since the fragment was not actually destroyed, the HomeViewModel was not cleared. This means when we go back to the HomeFragment (by pressing back in the newly launched activity), the HomeViewModel is retrieved from the ViewModelStore. Because of this the init functions is not called and no new viewModelScope is set. Because we still reference the old one, the scope is not active anymore. Since RandomGenerator is a singleton, the RandomGenerator is a single instance (per koin module).

This is probably not the desired behavior. To make this scenario work, one option is to use a factory for RandomGenerator. This means that every time an instance is obtained, a new instance is created. If this is not an issue, this is an option. If the RandomGenerator is used in more places and is stateless (besides the viewModelScope) this option wastes resources.

Because of the issues above a better solution is to not depend on a lateinit var but pass in the scope as an argument to the constructor.

## Use Koin parameters

In this scenario we are going to utilize [Koin Parameters](https://insert-koin.io/docs/reference/koin-core/injection-parameters/) to inject the viewModelScope in the RandomGenerator. Parameters are injected into the object when an instance of the object (RandomGenerator in our case) is obtained.

The definition of the RandomGenerator is as follows:

```kotlin
single { parametersHolder ->
    RandomGenerator(parametersHolder.get())
}
```
As seen in the above definition, the ParametersHolder is passed in which can be used to obtain values during injection time. To actually pass-in those parameters the injection looks like this:

```kotlin
val randomGenerator: RandomGenerator by inject() {
    println("injection: inject $viewModelScope")
    parametersOf(viewModelScope)
}
```

Here you can see the viewModelScope is injected in the RandomGenerator. Each time the instance is resolved, the viewModelScope is injected. But remember, RandomGenerator is still a single so let's find out if this solution works as expected. The output of the same scenario can be seen below:

```text
inject androidx.lifecycle.CloseableCoroutineScope@6ee6970 <-- viewModelScope is injected using params
scope is active true
Do something in com.example.myapplication.ui.home.RandomGenerator@4ab5fd0 scope = androidx.lifecycle.CloseableCoroutineScope@6ee6970

inject androidx.lifecycle.CloseableCoroutineScope@57d6159 <-- new activity is launched, a new viewModelScope is injected
scope is active true
Do something in com.example.myapplication.ui.home.RandomGenerator@4ab5fd0 scope = androidx.lifecycle.CloseableCoroutineScope@6ee6970 <-- 2. But, we still have the same instance of RandomGenerator which references the original viewModelScope

onCleared called <-- back press
scope is active true
Do something in com.example.myapplication.ui.home.RandomGenerator@4ab5fd0 scope = androidx.lifecycle.CloseableCoroutineScope@6ee6970 <-- same RandomGenerator with a reference to the original viewModelScope
```

Although this scenario works, it is not what we want since in step 2, the RandomGenerator is still the same instance (single) with the reference to the original viewModelScope. Although it seems as if a new viewModelScope was injected, there is not an actual new instance of RandomGenerator created since it is defined as a single.

This mitigation is the same as in the previous scenario, use factory for RandomGenerator. The reasons to not do this are also the same and so this might not be an appropriate solution.

## Use a custom scope

todo
