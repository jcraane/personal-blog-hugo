---
layout:     post

title:      "Integrating Firebase JWT token verification in Ktor - Part 1"
subtitle:   "Using Firebase authentication with Ktor"
author: Jamie Craane
date: 2024-12-11
description: "This blog post guides you through integrating Firebase JWT token verification in a Ktor server application, enhancing security by leveraging Firebase's authentication capabilities."
image: "/img/part1-firebase-ktor.png"
published: true
showtoc: false
tags:
- Firebase
- Kotlin
- Ktor

categories: [ Kotlin ]
URL: "/2024/12/11/firebase_jwt_ktor_part_1/"
---

# Integrating Firebase JWT token verification in Ktor - Part 1

As applications become more complex and security becomes a priority, choosing the right tools to handle authentication is crucial. Today, we'll explore how to integrate Firebase JWT (JSON Web Token) authentication in a Ktor server application. This blog post will cover the core setup and implementation details required to ensure secure handling of JWTs in a Ktor application.

## Introduction to Ktor and Firebase Authentication

[https://ktor.io](https://ktor.io/) is a lightweight framework for creating web applications in Kotlin, known for its simplicity and flexibility, you will find no magic here. [Firebase Authentication](https://firebase.google.com/docs/auth) provides backend services, easy-to-use SDKs, and ready-made UI libraries to authenticate users to your app. It supports authentication using passwords, phone numbers, and popular federated identity providers like Google, Facebook, and Twitter.

## Setting Up Ktor with Firebase Authentication

In a typical Ktor application, modules are defined to manage different aspects of the application lifecycle. Here’s a simplified module setup to illustrate the integration process:

```kotlin
fun Application.module() {
    install(Koin) {
        slf4jLogger(Level.INFO)
        modules(securityModule(environment))
    }

	val firebaseAuth by inject<FirebaseAuth>()

    install(Authentication) {
        jwt("auth-jwt") {
	        /**
	        The verifier function allows you to verify a token format and its signature. In our case, we use the FirebaseSDK to do the token verification.
	        */
            verifier(
                FirebaseJWTVerifier(
                    firebaseAuth = firebaseAuth,
                    audience = "your-audience-id"
                )
            )
            /**
            The validate function allows you to perform additional validations on the JWT payload. Check the credential parameter, which represents a JWTCredential object and contains the JWT payload. In the example below, we just create a JWTPrincipal because we do not require additional validation.
            */
            validate { credential ->
                JWTPrincipal(credential.payload)
            }
            /**
            The challenge function allows you to configure a response to be sent if authentication fails.
            */
            challenge { _, _ ->
                call.respond(
                    HttpStatusCode.Unauthorized,
                    "Token is not valid or has expired"
                )
            }
        }
    }
}

```

## Integrating Firebase and Configuring FirebaseAuth

Before we delve into verifying JWT tokens, it's essential to configure FirebaseAuth. In this example we use Koin to initialize all of our dependencies. This setup involves downloading a Firebase service account JSON file (The instructions to create a Firebase config file can be found [here](https://firebase.google.com/docs/admin/setup)) and initializing FirebaseApp:

```kotlin
fun securityModule(environment: ApplicationEnvironment) = module {
    single {
        val firebaseConfig = environment.config.property("firebase.config").getString()

        val options = FirebaseOptions.builder()
            .setCredentials(
                GoogleCredentials.fromStream(
	                ByteArrayInputStream(firebaseConfig.toByteArray(Charsets.UTF_8)
                ))
            )
            .build()

        FirebaseApp.initializeApp(options)
        FirebaseAuth.getInstance()
    }
}

```

## Creating FirebaseJWTVerifier

The `FirebaseJWTVerifier` is responsible for decoding and verifying the JWTs provided by Firebase by using the Firebase SDK:

```kotlin
class FirebaseJWTVerifier(
    private val firebaseAuth: FirebaseAuth,
    private val audience: String
) : com.auth0.jwt.interfaces.JWTVerifier {

    override fun verify(token: String): DecodedJWT {
        val parts = token.split(".")
		val header = parts[0]
		val payload = parts[1]

		val firebaseToken = firebaseAuth.verifyIdToken(token)
		return FirebaseDecodedToken(firebaseToken, audience, token, header, payload)
    }
}

```

Here, `FirebaseDecodedToken` is a custom class that implements `DecodedJWT` to provide functionalities such as extracting claims and the JWT header and payload. It is just a wrapper around the firebaseToken.

## Handling JWT in Ktor Routes

Once JWT authentication is set up, you can secure specific routes by wrapping them with the `authenticate` block in your Ktor routing setup:

```kotlin
routing {
    authenticate("auth-jwt") {
        get("/secured-endpoint") {
            call.respondText("This is a secured endpoint!")
        }
    }
    get("/open-endpoint") {
        call.respondText("This is an open endpoint!")
    }
}

```

The `/secured-endpoint` will require a valid JWT to access, while `/open-endpoint` will remain unsecured. This flexibility allows you to mix secure and open routes within the same application.

## Potential errors

- When configuring Ktor authentication maken sure the Authentication plugin is installed before the Routing plugin. Else the following error occurs:

```
Exception in thread "main" io.ktor.server.application.MissingApplicationPluginException: Application plugin AuthenticationHolder is not installed

```

## Conclusion

Integrating Firebase JWT authentication with a Ktor application enhances security while leveraging Firebase's robust authentication management capabilities. By following the steps in this article you have configured Ktor to validate Firebase JWT's by using the Ktor JWT plugin and the Firebase SDK.

In the next article we are going to look into integrating Firebase JWT with users in our own database by using custom tokens and Kotlin context receivers.

## References

- [https://github.com/Moxie-Mobile/ktor-firebase-jwt](https://github.com/Moxie-Mobile/ktor-firebase-jwt)
- https://ktor.io/
- [https://firebase.google.com/docs/auth](https://firebase.google.com/docs/auth)
- [https://jwt.io](https://jwt.io/)
