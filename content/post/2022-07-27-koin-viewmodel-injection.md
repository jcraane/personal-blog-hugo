---
layout:     post

title:      "Koin ViewModel Injection"
subtitle:   "Inject objects into Android ViewModel which depend on a CoroutineScope"
author: Jamie Craane
date: 2022-07-27    
description: "This post describes various methods to use dependent objects in an Android ViewModel which depend on the viewModelScope."
image: "/img/robotFactory.jpg"
published: true
showtoc: false
tags:
- Android
- iOS
- KMM

categories: [ PROCESS ]
URL: "/2022/07/27/koin_viewmodel_injection/"
---

Image credit: Dall-E 'Inside of a large factory assembling robots from parts. The scene is somewhat dark with neon lights. cyberpunk style.'

## Context

A ViewModel in Android (not to be confused with the term ViewModel in the MVVM architecture pattern) is a component scoped to the lifecycle
of another component, usually a fragment or an activity. To create maintainable code without becoming the ViewModel to large, the ViewModel delegates to other objects. When using
coroutines, those other objects might use suspending functions. Those functions are called from the ViewModel in a CoroutineScope. For a ViewModel this is the viewModelScope. When the ViewModel is cleared, the viewModelScope is cancelled.

The above scenario works in the case all dependent objects can mark all required functions as suspended. This might not always be the case. Those dependent objects might also be used elsewhere or coming from a library. In this case it might make sense to pass a coroutine scope when initializing those objects. This post describes several methods (also ones which do not work as expected) to accomplish this.

### Application scenario

To test all scenario's I created a minimal application with the following scenario (the basis of the application is the navigator sample from Android Studio). The different scenarios are implemented in separate branches. The link to the source code can be found in the introduction of each scenario.

The app defines three tabs: home, dashboard and notifications. The app consists of a MainActivity and three fragments, one for each tab. The notifications tab contains a button which launches the MainActivity again (pressing back finishes this instance of the MainActivity). The view models are scoped to the fragments.

The button in the home fragment calls the RandomGenerator which generates a random number and set this in a MutableStateFlow. The flow is collected in the HomeFragment and displayed in a TextView.

The diagram visualizes this specific flow:

![img.png](/img/posts/koin-injection-app-flow.png)

The following class diagram visualizes the structure of the app:

![img.png](/img/posts/koin-injection-app-structure.png)

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

The last line shows that the scope in RandomGenerator is not active anymore. This is because when we started a new activity, the original HomeFragment was not destroyed (only onDestroyView was called). Since the fragment was not actually destroyed, the HomeViewModel was not cleared. This means when we go back to the HomeFragment (by pressing back in the newly launched activity), the HomeViewModel is retrieved from the ViewModelStore. Because of this the init function is not called and no new viewModelScope is set. Because we still reference the old one, the scope is not active anymore. Since RandomGenerator is a singleton, the RandomGenerator is a single instance (per koin module).

This is probably not the desired behavior. To make this scenario work, one option is to use a factory for RandomGenerator. This means that every time an instance is obtained, a new instance is created. If this is not an issue, this is an option. If the RandomGenerator is used in more places and is stateless (besides the viewModelScope) this option wastes resources.

Because of the issues above a better solution is to not depend on a lateinit var but pass in the scope as an argument to the constructor.

## Use Koin parameters

Source code: https://github.com/jcraane/AndroidKoinInjectViewModelScope/tree/koin_inject_viewmodelscope_parameters

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
HomeViewModel inject androidx.lifecycle.CloseableCoroutineScope@6ee6970 <-- viewModelScope is injected using params
RandomGenerator scope is active true
RandomGenerator Do something in com.example.myapplication.ui.home.RandomGenerator@4ab5fd0 scope = androidx.lifecycle.CloseableCoroutineScope@6ee6970

HomeViewModel inject androidx.lifecycle.CloseableCoroutineScope@57d6159 <-- new activity is launched, a new viewModelScope is injected
RandomGenerator scope is active true
RandomGenerator Do something in com.example.myapplication.ui.home.RandomGenerator@4ab5fd0 scope = androidx.lifecycle.CloseableCoroutineScope@6ee6970 <-- 2. But, we still have the same instance of RandomGenerator which references the original viewModelScope

HomeViewModel.onCleared called <-- back press
RandomGenerator scope is active true
RandomGenerator Do something in com.example.myapplication.ui.home.RandomGenerator@4ab5fd0 scope = androidx.lifecycle.CloseableCoroutineScope@6ee6970 <-- same RandomGenerator with a reference to the original viewModelScope
```

Although this scenario works, it is not what we want since in step 2, the RandomGenerator is still the same instance (single) with the reference to the original viewModelScope. Although it seems as if a new viewModelScope was injected, there is not an actual new instance of RandomGenerator created since it is defined as a single.

This mitigation is the same as in the previous scenario, use factory for RandomGenerator. The reasons to not do this are also the same and so this might not be an appropriate solution.

## Use a custom scope

Source code: https://github.com/jcraane/AndroidKoinInjectViewModelScope/tree/koin_inject_viewmodelscope_customscope

You can think of a [scope](https://insert-koin.io/docs/reference/koin-core/scopes/) of a window in which a certain object exists. What we want to achieve is scope the RandomGenerator to the lifecycle of the ViewModel (between the creation till the 
onCleared is called). Koin supports two ways of defining custom scopes: String Qualified Scope and Type Qualitied Scope. In our case the Type Qualitied Scope is used.

The following code defines the scope in the Koin module:

```kotlin
scope<HomeViewModel> {
    scoped { parametersHolder ->
        RandomGenerator(parametersHolder.get())
    }
}
```

The RandomGenerator is defined as a scoped component in the scope of HomeViewModel. To actually use this scope in the viewmodel see the following code:

```kotlin
class HomeViewModel : ViewModel(), KoinScopeComponent {
    override val scope: Scope by lazy { createScope(this) }

    val randomGenerator: RandomGenerator by inject {
        println("injection: inject $viewModelScope")
        parametersOf(viewModelScope)
    }

    override fun onCleared() {
        super.onCleared()
        scope.close()
        println("injection: onCleared called")
    }
}
```

The scope is created with the createScope function as soon as an instance of the HomeViewModel is created. The by inject uses this custom scope.
The scope is closed in the onCleared of the HomeViewModel. When we run the application we see the following output:

```text
HomeViewModel inject androidx.lifecycle.CloseableCoroutineScope@44334e9
RandomGenerator scope is active true
RandomGenerator Do something in com.example.myapplication.ui.home.RandomGenerator@f095bc9 scope = androidx.lifecycle.CloseableCoroutineScope@44334e9 <-- New instance of RandomGenerator 

HomeViewModel inject androidx.lifecycle.CloseableCoroutineScope@de1c11e <-- New HomeViewModel with new viewModelScope
RandomGenerator scope is active true
RandomGenerator Do something in com.example.myapplication.ui.home.RandomGenerator@c96e9a9 scope = androidx.lifecycle.CloseableCoroutineScope@de1c11e <-- New instance of RandomGenerator with new scope
HomeViewModel.onCleared called <-- scope is closed

RandomGenerator scope is active true
RandomGenerator Do something in com.example.myapplication.ui.home.RandomGenerator@f095bc9 scope = androidx.lifecycle.CloseableCoroutineScope@44334e9 <-- Original RandomGenerator with viewModelScope belonging to initial HomeViewModel
```

To test the HomeViewModel see [Koin Testing](https://insert-koin.io/docs/reference/koin-test/testing/) which describes how to inject objects into tests.

The advantage of this solution is that the lifecycle of the RandomGenerator aligns with the lifecycle of the view model. It is slightly more complex to implement but I think this is worth the tradeoff.

## Recap              

There are cases in which an Android view model needs an object which requires a coroutine scope. There are several ways to initialize the coroutine scope from the view model on the dependent object. This
posts described three ways (non-exhaustive) todo this. 

1. Using a lateinit var and initialize
2. Using Koin parameters without custom scope
3. Using Koin parameters with a custom scope

For every scenario the implementation is described as well as the pro's and con's. Option 1 is simple to implement with the tradeoff that it is easy to forget things or make mistakes. Option 2 utilizes Koin parameters and can only be used if factory bean definition can be used. Option 3 utilizes a custom scope to scope the dependent object to the life cycle of the view model. My preference would be option 3 since it makes it explicit that the dependent object and the view model share a similar life cycle.

## Things to consider

- When passing the viewModelScope to dependent components, those components should not close this scope themselves
- If possible, use components with suspending functions which are called from the view model. The view model is then responsible for launching a coroutine using the viewModelScope
- Beware that a single in Koin is not actual a Singleton in the JVM but a singleton within the same Koin module. It best to keep those stateless
- When defining a single with parameters and that single is injected in multiple places, you always get the same instance even if the injected parameters change. The parameter is injected the first time the instance is retrieved
- If possible it is best to pass all parameters in a ViewModel in the constructor and don't use setter injection at all
