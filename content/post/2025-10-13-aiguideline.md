---
layout: post
title: "Bridging Human Intent and AI Code Generation with @AIGuideline"
subtitle: "A Novel Approach to Embedding Architectural Guidelines in Your Codebase"
author: Jamie Craane
date: 2025-10-13
description: "Learn how the @AIGuideline annotation helps AI assistants understand and respect your project's architectural decisions and coding patterns."
image: "/img/posts/aiguideline.png"
published: true
showtoc: true
tags:
- Kotlin
- AI-Assisted Development
- Software Architecture
- Code Quality

categories: [Kotlin, AI]
URL: "/2025/13/10/aiguideline/"
---

### Introduction

As AI-powered coding assistants become increasingly prevalent in software development, a new challenge emerges: how do we ensure that AI-generated code aligns with our project's specific architectural patterns, design principles, and coding standards?

Existing AI guidelines, for example junie/.guidelines exist for this purpose. However, they are often large and challenging to maintain. Depending on the project, they can also contain a lot of information. The larger the guidelines file, the harder for the AI to follow consistently. 

- How can we keep the guidelines short and to the point?
- How can we keep the guidelines consistent with the codebase over the long run? 
- How can we couple the guidelines with the code itself?

In this post, we introduduce @AIGuideline, a novel approach to embedding AI architectural guidance directly into your codebase.

### The Problem: AI Assistants Need Context

Modern AI coding assistants like GitHub Copilot, JetBrains AI Assistant, and others are remarkably capable at generating syntactically correct code. However, they often lack understanding of:

- **Architectural patterns** specific to your project (e.g., use case-driven design, repository patterns)
- **Technology choices** and their rationale (e.g., why Kotlin Exposed instead of JPA)
- **Cross-cutting concerns** like testing strategies or dependency injection frameworks
- **Project-specific conventions** that differ from general best practices

Without this context, AI-generated suggestions might technically work but violate your project's architectural principles, creating technical debt and inconsistency.

### The Solution: Self-Documenting Architecture

The `@AIGuideline` annotation provides a mechanism to embed architectural guidance directly where it matters most—in your code. Here's the implementation:

```kotlin
package nl.moxie.embeddings.common.annotations

/**
 * Annotation for documenting guidelines within the codebase.
 * 
 * @property description The description of the guideline.
 */
@Retention(AnnotationRetention.SOURCE)
@Target(
    AnnotationTarget.CLASS,
    AnnotationTarget.FUNCTION,
    AnnotationTarget.PROPERTY,
    AnnotationTarget.FILE,
    AnnotationTarget.VALUE_PARAMETER
)
annotation class AIGuideline(
    val description: String,
)
```

The annotation is intentionally simple:
- **Source-level retention**: No runtime overhead, purely for documentation
- **Broad target support**: Can be applied to classes, functions, properties, files, and parameters
- **Single parameter**: A concise description of the guideline

### Integrating with AI Assistant Guidelines

To ensure AI assistants recognize and respect these annotations, reference them in your project's AI guidelines file (typically `.junie/guidelines.md` or similar):

```markdown
## Backend Development

Important: Any code or section annotated with @AIGuideline indicates 
a project rule or practice that must be strictly followed by AI code 
generation or suggestions. Always prioritize and adhere to the logic, 
constraints, and recommendations present in such annotated code when 
generating, revising, or explaining code for this project. Do not add 
the @AIGuideline annotation yourself.
```

This explicit instruction ensures AI assistants:
1. **Recognize** the annotation as a signal of critical architectural information
2. **Prioritize** these guidelines when generating or suggesting code
3. **Respect** the boundaries—not adding annotations themselves but following existing ones

### Real-World Examples

Let's explore how `@AIGuideline` is used in a real Kotlin/Ktor application to communicate architectural patterns to AI assistants.

#### Example 1: Technology Choice Guidance

```kotlin
@AIGuideline("Database access is implementing using Kotlin Exposed")
class DocumentsExposedRepository : DocumentsRepository {
    // Implementation using Exposed SQL library
}
```

**What this communicates to AI:**
- The project uses Kotlin Exposed, not JPA or other ORMs
- When generating database access code, follow Exposed patterns
- Don't suggest alternative database libraries

#### Example 2: Testing Strategy

```kotlin
@AIGuideline("Time repository is implementing using Kotlin Time and should be used when the current time is needed to facilitate testing")
@OptIn(ExperimentalTime::class)
interface TimeRepository {
    fun now(): Instant
}
```

**What this communicates to AI:**
- Never use `System.currentTimeMillis()` or `Clock.systemUTC()`
- Always inject TimeRepository for time-dependent logic
- This pattern exists specifically to enable deterministic testing

#### Example 3: Abstraction Boundaries

```kotlin
@AIGuideline("This interface abstracts the searching of content from different technologies and sources.")
interface ContentSearcher {
    fun search(): List<Document>
}
```

**What this communicates to AI:**
- Content searching is abstracted to support multiple implementations
- When adding new search capabilities, implement this interface
- Don't couple search logic directly to specific technologies

#### Example 4: Dependency Injection Configuration

```kotlin
@AIGuideline("The main module is responsible for configuring the Koin dependency injection framework")
fun mainModule(environment: ApplicationEnvironment) = module {
    single<String>(named(OPEN_API_KEY)) {
        System.getenv("OPENAI_API_KEY")
            ?: throw IllegalStateException("OPEN_API_KEY environment variable is required")
    }
    
    single<DataSource> {
        val config = HikariConfig()
        // Configuration...
        HikariDataSource(config)
    }
    
    // Additional dependency configurations...
}
```

**What this communicates to AI:**
- The project uses Koin for dependency injection
- All DI configuration happens in this module function
- When adding new dependencies, register them here following Koin patterns

#### Example 5: API Encapsulation

```kotlin
@AIGuideline("The OpenAI API's are encapsulated in the OpenAIApi implementation")
interface OpenAIApi {
    suspend fun createEmbeddings(input: List<String>): EmbeddingsResponse
    suspend fun chatCompletion(messages: List<ChatMessage>): ChatCompletionResponse
}
```

**What this communicates to AI:**
- External API calls to OpenAI should go through this interface
- Don't make direct HTTP calls to OpenAI in other parts of the codebase
- This abstraction enables testing and potential API provider swapping

### Benefits of This Approach

#### 1. **Co-location of Intent and Implementation**
Guidelines live exactly where they're needed, not in a separate wiki or document that might become outdated.

#### 2. **Discoverability**
When an AI assistant analyzes a file, it immediately sees the architectural guidelines through these annotations.

#### 3. **Version Control**
Guidelines evolve with your code. When you refactor, the guidelines travel with the code.

#### 4. **Consistency in AI-Generated Code**
AI assistants can maintain architectural consistency even when generating code for complex scenarios they haven't seen before.

#### 5. **Onboarding Aid**
New developers (human or AI) can quickly understand architectural patterns by looking at annotated interfaces and base classes.

#### 6. **Simpler common guidlines**
Adding @AIGuidline minimizes the information required in .junie/guidelines.md. This makes it easier to maintain and update.

### Best Practices

#### 1. **Be Concise but Clear**
```kotlin
// Good: Clear and actionable
@AIGuideline("Use repository pattern for all database access")

// Too vague
@AIGuideline("Database stuff goes here")

// Too verbose
@AIGuideline("This repository should be used whenever you need to access the database because we've chosen to use the repository pattern as described in Clean Architecture by Robert Martin...")
```

#### 2. **Focus on Architectural Decisions**
Use `@AIGuideline` for decisions that affect code structure, not for trivial implementation details:

```kotlin
// Good: Architectural pattern
@AIGuideline("Validation logic is implemented as extension functions on input types")

// Unnecessary: Language convention
@AIGuideline("Use data classes for immutable data")
```

#### 3. **Apply at the Right Level**
- **Interfaces/Abstract Classes**: Define patterns and contracts
- **Top-Level Functions**: Establish project-wide utilities or conventions
- **Files**: Document module-level decisions

#### 4. **Don't Over-Annotate**
As mentioned in the guidelines: "Do not add the @AIGuideline annotation yourself." This instruction is for AI assistants—use the annotation judiciously for truly important architectural decisions, not every class or function.

#### 5. **Complement, Don't Replace Documentation**
`@AIGuideline` annotations work best alongside comprehensive documentation. Use annotations for quick, actionable guidance and detailed documentation for deeper explanation.

### Conclusion

The `@AIGuideline` annotation represents a paradigm shift in how we communicate architectural intent. By embedding guidelines directly in our codebase, we create a self-documenting architecture that both human developers and AI assistants can understand and respect.

As AI-assisted development continues to evolve, techniques like this will become increasingly important. The code becomes not just a set of instructions for computers, but also a comprehensive guide for AI assistants to generate contextually appropriate, architecturally sound code.

Whether you're working with AI pair programmers today or preparing for tomorrow's development workflows, `@AIGuideline` offers a simple, elegant way to ensure your architectural vision is preserved and propagated throughout your codebase.

The `@AIGuideline` annotation is language-agnostic in concept. While the examples here are in Kotlin, the same approach can be adapted to other languages.

### References

- [Kotlin Annotations Documentation](https://kotlinlang.org/docs/annotations.html)
- [Clean Architecture Principles](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Self-Documenting Code Best Practices](https://martinfowler.com/bliki/CodeAsDocumentation.html)
