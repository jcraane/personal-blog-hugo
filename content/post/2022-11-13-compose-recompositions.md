---
layout:     post

title:      "Jetpack Compose - Recompositions"
subtitle:   "Recompositions in Jetpack Compose"
author: Jamie Craane
date: 2022-11-13
description: "This post describes what Compose recomposition is and how to optimize your Compose code to minimize recompositions."
image: "/img/recompositions-header.png"
published: false
showtoc: false
tags:
- Android
- Compose

categories: [ Android ]
URL: "/2022/11/13/compose_recompositions/"
---

# What is recomposition

A Compose function in Jetpack Compose represents some UI to be rendered:

```kotlin
@Composable
fun UsersScreen() {
    var usersScreenState by remember {
        mutableStateOf(
            UsersScreenState(
                title = "Users",
                body = "This is the users screen",
                (1..40).toList().map { Person(it, "Firstname Lastname $it") },
            )
        )
    }

    Column() {
        Text(usersScreenState.title)
        Text(usersScreenState.description)

        Button(onClick = { usersScreenState = usersScreenState.copy(title = Random.nextInt().toString()) }) {
            Text("Change title")
        }

        PersonList(usersScreenState.persons)
    }
}
```

Notice that the usersScreenState is a var which means it can change. Any time the usersScreenState changes, the UI is recomposed. This is 
also called recomposition. Smart recomposition means skipping Composables when their inputs have not changed and those inputs are considered stable.

The part 'when inputs change' is important. This applies that when inputs are not changed, Composables are not re-rendered. This is indeed
the case, according to the docs: When Compose recomposes based on new inputs, it only calls the functions or lambdas that might have changed, and skips the rest.

The UI of the app looks like this:

![img.png](/img/posts/recomposition-app.png)

# Why does it matter

It is important for performance reasons that a Composable can be skipped if its inputs have not changed. A Composable can be executed many
times. When you have an animation, Composables can be executed many times. To getter a stutter free UI, it is important that portions
of the UI which have not change are not re-rendered.

# How does Compose determine that a Composable might be skipped

Look again the definition of recomposition: Smart recomposition means skipping Composables when their inputs have not changed and those inputs are considered stable.

How does Compose consider a certain type stable? To be a stable type, the type must adhere to the following properties:

- Call to equals returns the same value for the same instances.
- Composition is notified when a public property of the type changes.
- All public properties are primitive types or also considered stable.

All primitives are considered stable by default. 

# List type and recomposition

Going back to the above example. When the Change title button is pressed, the title of the usersScreenState is updated with a random value. 
Based on our intuition we might think that only the Title is recomposed when usersScreenState changes. This is not the case however!

What actually happens is that the whole PersonList is also recomposed along with the Composables in it. We can visualize this by
using the Layout inspector in Android Studio. The Layout Inspector shows the UI Tree along with how many times Composables are recomposed or skipped.

![img.png](/img/posts/recomposition-layout-inspector.png)

As you can see in the above screenshot, the Title is recomposed and the Description is skipped. This is expected because the title has changed
whereas the description has not. What we also see is that all elements in the LazyColumn are also recomposed. This is not what we expect because
the person list has not changed.

# How to fix

How can we fix this? Take a look at the third requirement for Compose to consider a type stable: All public properties are primitive types or also considered stable.

The persons property is of the following type:

```kotlin
 val persons: List<Person>
```

As you can see the type is List<Person>. Since List is an interface, Compose cannot determine if this is a mutable or immutable implementation. 
Therefore, the UsersScreenState data class is considered unstable. As mentioned in [Compose Metrics](https://chris.banes.dev/posts/composable-metrics/) we can take a look at the class stability and notice the following:

```plain
unstable class UsersScreenState {
  stable val title: String
  stable val body: String
  unstable val persons: List<Person> <-- List<Person> considered unstable
  <runtime stability> = Unstable
}
stable class Person {
  stable val id: Int
  stable val name: String
  <runtime stability> = Stable
}
```

As expected, the List<Person> type is considered unstable and so is the enclosing type UsersScreenState. 

To fix this we can use the Immutable annotation to inform the Compose compiler that all public properties of the type remain unchanged after creation. 
We cannot use this annotation on an individual property. We need to create a wrapper class for the list of persons:

```kotlin
data class UsersScreenState(
    val title: String,
    val body: String,
    val personCollection: PersonCollection,
)

@Immutable
data class PersonCollection(val persons: List<Person>)
```

By introducing a wrapper class and annotate that as @Immutable, the UsersScreenState becomes stable. We can proof this by looking at the class
metrics:

```plain
stable class UsersScreenState {
  stable val title: String
  stable val body: String
  stable val personCollection: PersonCollection <- PersonCollection is table so UsersScreenState is also considered stable
  <runtime stability> = Stable
}
stable class PersonCollection {
  unstable val persons: List<Person>
}
```

Finally, to see if it works, lets re-run the layout inspector to see if we get rid of the recompositions in the list:

![img.png](/img/posts/recomposition-layout-inspector-good.png)

The list is indeed not recomposed because the PersonCollection has not changed.

# Conclusion

When working with Jetpack Compose it is important to know how certain code may affect performance. One of the aspects is recomposition. 
Recomposition means execute Composable functions again when inputs have changed. Smart recompositions means skipping those Composable
functions when inputs have not changed.

Compose uses certain properties anout types to determine if a type is considered stable. Only with a stable type Compose is able to 
skip a Composable if inputs have not changed.

For certain types Compose cannot determine if a type is stable, for example List. This is because List is an interface which can have
both a mutable or immutable implementation. To help Compose with stability you can use the Immutable annotion on the class level.

Compose has various tools to inspect the stability and number of recomposed or skipped composables.

With this post you now have the knowledge to optimize Compose recompositions.
