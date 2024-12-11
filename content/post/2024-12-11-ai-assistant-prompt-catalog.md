
---
layout:     post

title:      "AI Assistant Prompt Catalog"
subtitle:   "Collection of useful prompts for Jetbrains AI Assistant"
author: Jamie Craane
date: 2024-12-11
description: "This blog post provides a collection of powerful prompts for JetBrains AI Assistant, designed to boost productivity and simplify development tasks. Perfect for Kotlin developers, it demonstrates how to leverage AI effectively within JetBrains IDEs."
image: "/img/custom-prompts.png"
published: true
showtoc: false
tags:
- Kotlin
- AI

categories: [ Kotlin ]
URL: "/2024/12/11/ai-assistant-prompt-catalog/"
---

# AI Assistant prompt catalog

## Prompts

### Android: Replace kotlin synthetic view viewBinding

@author: https://x.com/jcraane

Kotlin synthetics are deprecated. This prompt replaces Kotlin synthetics in a file with viewBinding:

```text
Replace kotlin synthetic with viewBinding.

Prefix all Koltin synthetic views with binding.

Replace snake case view names with camel casing.

$SELECTION
```

### Create a unit test using Kotest and BehaviorSpec

@author: https://x.com/jcraane

```text
You are an expert AI programmer which uses unit tests to verify behavior.

- Use Kotlin kotest BehaviorSpec
- Write unit tests for the obvious cases
- Write unit tests for the edge cases
- DO NOT EXPLAIN THE TEST
    
$SELECTION
```

### Create a unit test using Jest

@author: https://x.com/jcraane

```text
Write a unit test using Jest and typescript.

- Write unit tests for the obvious cases
- Write unit tests for the edge cases
- DO NOT EXPLAIN THE TEST
    
$SELECTION
```

### Document API route

@author: https://x.com/jcraane

```text
Write OpenApi docs for all routes which miss documentation. Write the documentation as Kotlin kdoc in the code, use the following as an example:

        /**
         * GET /v1/users/profile
         *
         * Retrieves the profile of the currently authenticated user.
         * Requires the user to be logged in.
         *
         * Response:
         * - 200 OK: Returns the user's profile information.
         * - 401 Unauthorized: The user is not authenticated.
         */
        get("/users/profile") {
            val user = call.getAuthenticatedUser()
            val profile = userService.getUserProfile(user.id)
            call.respond(HttpStatusCode.OK, profile)
        }

- Make the documentation user-friendly

$SELECTION  
```

### Review code

@author: https://x.com/jcraane

```text
You are an expert AI programmer reviewing code written by others. During the review:

        - Summarize what is good about the code
        - Mention where the code can be improved and why if applicable
        - Check if there are potential security issues
        - Check if there are potential performance issues
        - Check for potential duplicate code and suggest refactorings
        - DO NOT EXPLAIN THE CODE
    
 $SELECTION
```
