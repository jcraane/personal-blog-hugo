---
layout:     post

title:      "Integrating Firebase auth in server side Thymeleaf app"
subtitle:   "Using Firebase authentication with Ktor and Thymeleaf"
author: Jamie Craane
date: 2025-01-15
description: "This post, we will look at how to integrate your Firebase users in your own application database."
image: "/img/part1-firebase-ktor.png"
published: false
showtoc: false
tags:
- Firebase
- Kotlin
- Ktor

categories: [ Kotlin ]
URL: "/2025/06/06/firebase_auth_ktor_thymeleaf/"
---

# Server-Side Login with Firebase in Thymeleaf Templates

This blog post explains how to implement server-side authentication with Firebase in a Kotlin Ktor application using Thymeleaf templates. This approach is particularly useful for admin panels or dashboards where you want to leverage Firebase Authentication while rendering server-side HTML. By leveraging Thymeleaf (or other server-side templates for example Freemarker), may shorten the time-to-market for these kind of dashboard compared to full fledged Javascript frameworks like Angular or React.

## Introduction

In [previous posts](https://jamiecraane.dev/2024/12/11/firebase_jwt_ktor_part_1/), we explored how to implement Firebase authentication in Ktor for client applications. In this post, we'll focus on server-side rendering with Thymeleaf templates while still using Firebase for authentication.

This approach offers several benefits:
- Secure server-side session management
- Traditional web application flow with form submissions
- Leveraging Firebase's authentication capabilities
- No need for client-side JavaScript authentication frameworks or libraries

## Project Setup

First, let's set up our dependencies in the `build.gradle.kts` file:

```kotlin
dependencies {
    // Ktor core dependencies
    implementation(libs.ktor.server.core)
    implementation(libs.ktor.server.netty)
    
    // Authentication
    implementation(libs.ktor.server.auth)
    implementation(libs.ktor.server.auth.jwt)
    implementation(libs.ktor.server.sessions)
    
    // Templating
    implementation(libs.ktor.server.thymeleaf)
    
    // Firebase Admin SDK
    implementation(libs.firebase.admin)
}
```

## Configuring Thymeleaf

We need to configure Thymeleaf to handle our templates:

```kotlin
fun Application.configureTemplating() {
    install(Thymeleaf) {
        setTemplateResolver(
            ClassLoaderTemplateResolver().apply {
                prefix = "templates/"
                suffix = ".html"
                templateMode = TemplateMode.HTML
                characterEncoding = "utf-8"
            }
        )
        
        routing {
            // Public route for login page
            get("/admin/login") {
                call.respond(ThymeleafContent("admin/login", emptyMap()))
            }
            
            // Protected routes that require authentication
            authenticate(ADMIN_AUTH) {
                get("/admin/dashboard") {
                    call.respond(ThymeleafContent("admin/dashboard", emptyMap()))
                }
            }
        }
    }
}
```

For more information about configuring Thymeleaf with Ktor see: https://ktor.io/docs/server-thymeleaf.html#use_template

## Creating the Login Template

Our login page is a simple HTML form that posts to our authentication endpoint:

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Admin Login</title>
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-gray-100 font-sans">
<div class="min-h-screen flex items-center justify-center">
    <div class="bg-white p-8 rounded-lg shadow-md w-full max-w-md">
        <div class="text-center mb-8">
            <h1 class="text-2xl font-bold text-gray-800">Admin Panel</h1>
            <p class="text-gray-600">Please sign in to access the admin panel</p>
        </div>

        <form action="/admin/login" method="post" class="space-y-6">
            <!-- Error message display -->
            <div th:if="${error}" class="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded relative">
                <span th:text="${error}">Invalid credentials. Please try again.</span>
            </div>

            <!-- Success message display -->
            <div th:if="${message}" class="bg-green-100 border border-green-400 text-green-700 px-4 py-3 rounded relative">
                <span th:text="${message}">Success message</span>
            </div>

            <div>
                <label for="email" class="block text-sm font-medium text-gray-700">Email</label>
                <input type="email" id="email" name="email" required
                       th:value="${email}"
                       class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500">
            </div>

            <div>
                <label for="password" class="block text-sm font-medium text-gray-700">Password</label>
                <input type="password" id="password" name="password" required
                       class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500">
            </div>

            <div>
                <button type="submit"
                        class="w-full flex justify-center py-2 px-4 border border-transparent rounded-md shadow-sm text-sm font-medium text-white bg-blue-600 hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-blue-500">
                    Sign in
                </button>
            </div>
        </form>
    </div>
</div>
</body>
</html>
```

## Implementing the Login Route

Now, let's implement the route that handles the login form submission:

```kotlin
fun Route.adminRoutes(
    firebaseAuth: FirebaseAuth,
    userService: UserService,
    firebaseApi: FirebaseApi,
) {
    route("/admin") {
        post("/login") {
            val parameters = call.receiveParameters()
            val email = parameters["email"] ?: return@post call.respond(
                HttpStatusCode.BadRequest,
                ErrorResponse("Email is required")
            )
            val password = parameters["password"] ?: return@post call.respond(
                HttpStatusCode.BadRequest,
                ErrorResponse("Password is required")
            )

            runCatching {
                // Step 1: Sign in with Firebase using email/password
                val signInResponse = signInWithEmailAndPassword(email, password, firebaseApi)
                val firebaseUid = signInResponse.localId
                
                // Step 2: Get the user from our database and verify they're an admin. This function throws
               // an exception if the user is not found or does not have the ADMIN role.
                val user = getAuthenticatedAdmin(userService, firebaseUid)
                
                // Step 3: Set custom claims for the user. This makes sure the roles (with ADMIN) will be present in the Firebase idToken
                setCustomClaimsForUser(user, firebaseAuth, firebaseUid)
                
                // Step 4: Generate a session cookie
                val sessionCookie = generateSessionCookie(firebaseAuth, signInResponse)
                call.sessions.set("session", sessionCookie)
                
                // Step 5: Redirect to dashboard
                call.respondRedirect("/admin/dashboard")
            }.onFailure {
                call.respond(
                    HttpStatusCode.Unauthorized,
                    ErrorResponse("Invalid credentials")
                )
            }
        }

        post("/logout") {
            runCatching {
                call.sessions.clear("session")
                call.respondRedirect("/admin/login")
            }.onFailure {
                call.respond(
                    HttpStatusCode.InternalServerError,
                    ErrorResponse("Logout failed: ${it.message}")
                )
            }
        }
    }
}
```

Let's break down the key helper functions:

```kotlin

private suspend fun signInWithEmailAndPassword(
   email: String,
   password: String,
   firebaseApi: FirebaseApi,
): SignInWithPasswordResponse {
   // Call Firebase REST API to authenticate with email/password
   val signInRequest = SignInWithPasswordRequest(
      email = email,
      password = password,
      returnSecureToken = true
   )

   return firebaseApi.signInWithPassword(signInRequest)
}

private fun generateSessionCookie(
    firebaseAuth: FirebaseAuth,
    signInResponse: SignInWithPasswordResponse,
): String? {
    // Create a session cookie that expires in 5 days (or user whatever expiry is sufficient for your use case)
    val expiresIn: Long = TimeUnit.DAYS.toMillis(5)
    val options = SessionCookieOptions.builder()
        .setExpiresIn(expiresIn)
        .build()

    // Create the session cookie using the ID token from Firebase
    return firebaseAuth.createSessionCookie(signInResponse.idToken, options)
}

private fun setCustomClaimsForUser(
    user: User,
    firebaseAuth: FirebaseAuth,
    firebaseUid: String,
) {
    // Set custom claims for the user, including role information so this information is present in the token.
    // This way, we do not need to retrieve the user with every request to determine the roles of the user
    val customClaims = mapOf(
        NATIVE_USER_ID_CLAIM to user.id.toString(),
        TIMEZONE_ID_CLAIM to user.data?.timeZoneId,
        ROLES_CLAIM to UserRole.ADMIN.code
    )

    firebaseAuth.setCustomUserClaims(firebaseUid, customClaims)
}
```

## The Firebase API Interface

Please note firebaseApi.SignInWithPasswordRequest is used to login wht email/password server-side. This functionality is not presenet in the firebase admin sdk. Therefore
a firebaseApi is created which leverages the Firebase REST API to provide this functionality. The implementation uses Ktor client to call the Firebase Rest API. 


## Configuring Firebase Authentication

Since sessions are used to manage authenticated users, we need to setup session authentication which validates the session
cookie and the required roles of the user which access the /admin pages.

```kotlin
fun Application.configureJwtAuth() {
    val firebaseAuth by inject<FirebaseAuth>()

    install(Sessions) {
        cookie<String>("session")
    }

    install(Authentication) {
        session<String>(ADMIN_AUTH) {
            validate { sessionCookie ->
                try {
                    // Verify the session cookie
                    val decodedToken = firebaseAuth.verifySessionCookie(sessionCookie, true)
                    val roles = decodedToken.claims["roles"] as? String
                    
                    // Check if user has admin role
                    if (roles == UserRole.ADMIN.code) {
                        UserIdPrincipal(decodedToken.uid)
                    } else {
                        null
                    }
                } catch (e: Exception) {
                    null
                }
            }
        }
    }
}
```

## Dashboard Template

Once authenticated, users are redirected to the dashboard:

```html
<!DOCTYPE html>
<html lang="en" th:replace="~{layout :: layout(~{::content}, ~{::title}, ~{::breadcrumb})}">
<head>
    <title th:fragment="title">Admin Dashboard</title>
</head>
<body>
<!-- This content will be injected into the layout -->
<div th:fragment="content">
    <h1 class="text-2xl font-bold mb-6">Welcome to the Admin Dashboard</h1>
    
    <!-- Dashboard content here -->
</div>

<!-- This will be used by the layout for the breadcrumb -->
<span th:fragment="breadcrumb">Dashboard</span>
</body>
</html>
```

## How It All Works Together

1. The user navigates to `/admin/login` and sees the login form
2. They enter their email and password and submit the form
3. The server calls Firebase to authenticate the credentials
4. If successful, the server:
    - Verifies the user has admin privileges in our database
    - Sets custom claims for the user in Firebase
    - Creates a session cookie with Firebase
    - Sets the session cookie in the browser
    - Redirects to the dashboard
5. For subsequent requests, the session cookie is validated against Firebase
6. When the user logs out, the session is cleared

## Security Considerations

- The Firebase server sdk has no function for signing in with email and password. We need to leverage the Firebase REST API for this
- Firebase handles the creation of session cookies using the Firebase admin SDK
- Server-side validation ensures only authorized users can access protected routes
- Custom claims allow for role-based access control. Custom claims are persisted using the setCustomUserClaims fuction of the Firebase SDK

## Conclusion

Using Firebase Authentication with server-side Thymeleaf templates gives you the best of both worlds: Firebase's robust authentication system and the simplicity of server-rendered HTML. This approach is perfect for admin panels or dashboards where you want to leverage Firebase while maintaining a traditional web application flow.

## References

- https://ktor.io/docs/server-thymeleaf.html
- https://ktor.io/docs/server-session-auth.html
- https://www.thymeleaf.org/doc/tutorials/3.1/usingthymeleaf.html#introducing-thymeleaf
- https://firebase.google.com/docs/reference/rest/database
