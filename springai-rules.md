# Airline Loyalty Assistant - Updated Implementation Rules

## 1. Language, Framework, and Build - FUNDAMENTAL PRINCIPLES

This application is a LOCAL DEMO.
Simplicity is the PRIMARY GOAL - DELIBERATELY IGNORE production best practices when they conflict with simplicity.

- Use Kotlin as the primary language for all new code
- Use Gradle as the build tool, with the Gradle wrapper and Kotlin DSL
- Use `gradle wrapper` command to generate wrapper files (DO NOT create wrapper files manually)
- Use Spring Boot as the application framework with minimal features
- DO NOT implement unnecessary features (health checks, metrics, advanced security)
- Use Spring AI for all LLM, RAG, and AI integration tasks
- UI should be simple, modern, and serve as static assets.
- DO NOT use any JavaScript frameworks for the UI (use simple html, css, and js)
- Security: Never hardcode secrets; use environment variables for sensitive configuration.
- Code Quality: Avoid deprecated APIs in Java/Kotlin.
- Makefiles: Use Makefiles (with emoji and ASCII colors) for multi-step commands.

### Framework

- Use the latest particular Spring Boot and Spring AI version. DO NOT change versions unless explicitly approved.
  - Spring Boot: 3.4.5-SNAPSHOT
  - Spring Dependency Management plugin: 1.1.7
  - Spring AI: 1.0.0-M7
  - Kotlin (JVM + Spring Plugin): 1.9.25
  - Java toolchain: Java 17
- Dependency Management: Use BOMs for Spring dependencies to ensure compatibility.

### Dependencies (Hard Requirements):

```kotlin
extra["springAiVersion"] = "1.0.0-M7"

dependencies {
  implementation("org.springframework.boot:spring-boot-starter-web")
  implementation("org.springframework.ai:spring-ai-starter-model-openai")
  implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
  implementation("org.jetbrains.kotlin:kotlin-reflect")
  testImplementation("org.springframework.boot:spring-boot-starter-test")
  testImplementation("org.jetbrains.kotlin:kotlin-test-junit5")
  testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}

dependencyManagement {
  imports {
    mavenBom("org.springframework.ai:spring-ai-bom:${property("springAiVersion")}")
  }
}
```

Use snapshots and milestones repositories for plugins and dependencies.
Use these exact versions unless a newer, stable release is explicitly approved.

## 2. Project Structure - CRITICAL GUIDELINES

- MAXIMUM 5-6 Kotlin files total for the core implementation
- Single controller for all endpoints - NO separate controllers per feature
- Single service layer for business logic
- When in doubt, choose the simpler solution over the "cleaner" architecture

## 3. Spring AI Integration - MANDATORY RULES
- **ALWAYS CHECK OFFICIAL DOCUMENTATION FIRST**: Before writing any Spring AI code, consult the [official documentation](https://docs.spring.io/spring-ai/reference/1.0/index.html) for the specific version being used (1.0.0-M7).
- Use the latest compatible Spring AI starter and BOM.
- Prefer auto-configuration for AI clients; only define beans if necessary.
- Use the recommended property structure for AI model configuration (e.g., `spring.ai.openai.api-key`)
- **VERIFY API SIGNATURES**: Package structures and method signatures in Spring AI may change between versions. Always verify you're using the correct imports and method signatures.
- **MODEL AGNOSTIC IMPLEMENTATION**: Design the integration to work with any LLM provider supported by Spring AI, not just a specific provider.
- **RESEARCH COMMON PATTERNS**: Research and understand the common patterns for Spring AI before implementing, particularly for:
  - Model initialization
  - Message formatting and sending
  - Response handling
- **INVESTIGATE ERRORS**: If encountering build errors, check package imports first, as they're the most common issue with Spring AI integration.
- Consult the official documentation and examples at https://docs.spring.io/spring-ai/reference/1.0/index.html

## Code Quality & Configuration

- Don't overcomplicate, avoid unnecessary abstractions
- Write idiomatic, concise, and readable code
- Anticipate future extensibility (RAG, Advisors, local models, new endpoints)
- Externalize configuration via `application.yml` or environment variables
- Limit configuration to only what's necessary for the demo

## Error Handling and Logging
- Implement MINIMAL but sufficient error handling for demonstration purposes
- Display errors directly in the template
- Focus on user-facing error messages over complex error infrastructure
- Log errors clearly for debugging but avoid complex logging frameworks
- Log all incoming requests and errors to the console for traceability

## Deliverables and Documentation
- README must document: Setup, environment variables, and how to run/test
- Deliverables must include: backend, frontend, build files, and README

## Zero-Debugging, Demo-Ready Delivery
- Deliver code that is robust and demo-ready in a single step
- Focus on making it WORK rather than making it architecturally perfect
- The demo should start successfully with a single command (./gradlew bootRun)
