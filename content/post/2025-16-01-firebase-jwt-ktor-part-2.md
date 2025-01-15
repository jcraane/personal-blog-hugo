---
layout:     post

title:      "Integrating Firebase JWT token verification in Ktor - Part 2"
subtitle:   "Using Firebase authentication with Ktor"
author: Jamie Craane
date: 2025-01-15
description: "This post, we will look at how to integrate your Firebase users in your own application database."
image: "/img/part1-firebase-ktor.png"
published: true
showtoc: false
tags:
- Firebase
- Kotlin
- Ktor

categories: [ Kotlin ]
URL: "/2025/01/16/firebase_jwt_ktor_part_2/"
---

# Part 2: Integrating Firebase Users into Your Database with Ktor

In the first part of this blog series, we explored how to integrate Firebase JWT authentication into your Ktor application. Besides creating accounts and logging in, we also want to store a reference of the Firebase user in our own database. This way, data can be associated with a specific user.

In this post, we will look at how to integrate your Firebase users in your own application database. This article will cover using **custom tokens**, **context receivers**, and the **Exposed SQL library** to implement this functionality.

## Introduction to Custom Tokens and User Management in Firebase

In many cases, storing users in your own database is required to track additional user attributes and metadata beyond what Firebase offers out of the box.

But how can we efficiently associate a Firebase user with a user  in our own database? By efficient I mean, not checking the database with every request to verify if the Firebase user is present or not.

One way of doing this, is leveraging Firebase custom tokens.

## Setting Up the User Model and Database

In our example, we’ve used PostgreSQL as the database backend and the **Exposed SQL library** to interact with the database. Below is a sample schema for managing users.

### Creating the User Table

A firebase_id columns is added where the Firebase uid is stored. Another option is creating a separate table containing the external id’s of the user. In that case you trade in **simplicity** for **flexibility**.

```sql
CREATE TABLE users
(
    id            uuid PRIMARY KEY,
    firebase_id   VARCHAR(36) UNIQUE,
    first_name    VARCHAR(150) NULL,
    last_name     VARCHAR(150) NULL,
    email         VARCHAR(150) UNIQUE,
    created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_modified TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

```

The idea is that we associate Firebase users via de uid to the users in the database. However, the backend does not expose the Firebase uid. Instead, the native id of the user is exposed in the form of a typed id, like so: user_01jhk53fajfbyb9abhks1bp8yx.

How do we obtain the native user id when a user makes a request to our backend?

### Leveraging Custom Tokens

Firebase has the option to authenticate users and devices using JSON Web Tokens (JWT). The backend generates these token and can add additional attributes. This is exactly with the create-custom-token endpoint does.  It receives a (valid) Firebase token set our native user id as custom attribute on the token.

In this stage, the user is created if the user did not already exists in our own database, together with the firebase uid. Below is the sample implementation:

```kotlin
// The create-custom-token endpoint is secured by verifying the Firebase token with which this endpoint is called
authenticate("auth-jwt-custom-token") {
	post("/create-custom-token") {
    val jwtPrincipal = call.principal<JWTPrincipal>()
    val firebaseId = jwtPrincipal?.subject ?: return@post call.respond(HttpStatusCode.Unauthorized)

    // Fetch or create user
    val user = userService.retrieveUserByFirebaseId(firebaseId)
        ?: createUser(userService, jwtPrincipal, firebaseId)

// Return the custom token in the response to the client
    call.respondText(createCustomTokenForUser(firebaseAuth, user))
	}
}
```

The custom token itself is created using the Firebase admin SDK:

```kotlin
private fun createCustomTokenForUser(firebaseAuth: FirebaseAuth, user: User): String {
    return firebaseAuth.createCustomToken(
	    user.firebaseId, 
	    mapOf("nativeUserId" to user.id.toString()), // 'Our' userId is set as custom attribute on the token
	  )
}

```

Remember, the create-custom-token endpoint is called only when the user logs in to our application. The payload of a custom token cannot exceed 1000 bytes. The following sequence diagram shows the create-custom-token flow:

![img.png](/img/posts/create-custom-token.png)

The following is an example (some details omitted) of how the custom attribute is present in the custom token:

```jsx
{
  "claims" : {
    "nativeUserId" : "user_01jhk53ddsfbyb9qvwks1bpabc"
  },
  "iat" : 1736971186,
  "uid" : "mzetIdaas1Se9qBdkskJGxMcson2",
  "sub" : "firebase-adminsdk",
  "aud" : "https:\/\/identitytoolkit.googleapis.com\/google.identity.identitytoolkit.v1.IdentityToolkit",
  "exp" : 1736974786,
  "iss" : "firebase-adminsdk"
}
```

## Using Context Receivers for Authenticated User Context

Now that our native user is associated with a Firebase user, how do we obtain the currently authenticated user in our application logic?

For this, we can use  **Kotlin** **context receivers. Instead** of passing the authenticated user down through function parameters, **Kotlin** **context receivers** simplify access to the current authenticated user. Here’s an example of how this works when handling tags for the authenticated user.

### Example: Adding Tags for an Authenticated User

In this implementation, a tag object is linked to the authenticated user stored in a database. this funciton uses  **context receivers** to obtain the authenticated user without passing this user explicitly through function parameters.

```kotlin
context(AuthenticatedUser)
override suspend fun addTag(name: String, documentId: DocumentId?): Tag {
    val tag = Tag(typeId = typeId, name = name, created = timeRepository.now())

    newSuspendedTransaction {
        TagsTable.insert {
            it[id] = tag.id.uuid
            it[TagsTable.name] = tag.name
            it[created] = tag.created
            it[userId] = nativeUserId.uuid
        }
    }

    return tag
}

```

In the above example, the nativeUserId is a property in the Authenticated user.

## Implementing the route

The final piece of the puzzle is implementing the route which calls our service/repository. This also means setting the context so the authenticated user is passed with each function call.

The Ktor `authenticate` block is used to ensure specific routes are secured. These routes rely on the `JWTPrincipal` to provide the authenticated user's context.

For example:

```kotlin
route("v1/tags") {
    authenticate("auth-jwt") {
        post {
            val body = call.receive<CreateTagRequest>()
            // The with function defines the context which means the autenticated user is passed via a context receiver. If this is omitted, a compile error appears 
            val response = with(call.principal<JWTPrincipal>()!!.authenticatedUser(typeId)) {
                body.tags.map {
                    tagService.addTag(name = it.name)
                }
            }
            call.respond(HttpStatusCode.OK, response)
        }
    }

```

You might ask yourself the question, what about the two different configurations for verifying JWT tokens?

1. authenticate("auth-jwt") is for all routes which need to verify our custom token
2. authenticate(”auth-jwt-custom-token”) is specifically for the create-custom-token, which validates the token when the user first logs in to Firebase. If this token cannot be verified, we do not create a custom token.

## Conclusion

In this blog post, we explored integrating Firebase users into your database using Ktor, focusing on associating Firebase users with native database entries for enhanced user attribute tracking.

- We discussed leveraging Firebase custom tokens to associate Firebase users with our own users in the database.
- The use of Kotlin's context receivers was highlighted to simplify accessing the authenticated user within application logic, improving code readability and maintainability.

By following these strategies, you can establish a robust user management system that effectively bridges Firebase authentication with your application's data needs.

In the next part of this series, we will look at integrating Firebase authentication in a Nextjs/React app leveraging client and server side authentication.

## References

- [https://firebase.google.com/docs/auth/admin/create-custom-tokens](https://firebase.google.com/docs/auth/admin/create-custom-tokens)
- [https://firebase.google.com/docs/auth/admin/custom-claims](https://firebase.google.com/docs/auth/admin/custom-claims)
- https://github.com/Kotlin/KEEP/blob/master/proposals/context-receivers.md
- https://github.com/JetBrains/Exposed
