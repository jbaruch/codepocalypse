# Airline Loyalty Assistant - Updated Implementation Rules

## 1. Language, Framework, and Build - FUNDAMENTAL PRINCIPLES
This application is a LOCAL DEMO. Simplicity is the PRIMARY GOAL - DELIBERATELY IGNORE production best practices when they conflict with simplicity.
- Use Kotlin as the primary language for all new code
- Use Gradle as the build tool, with the Gradle wrapper and Kotlin DSL
- Use `gradle wrapper` command to generate wrapper files
- Use Quarkus as the application framework with minimal features
- DO NOT implement unnecessary features (health checks, metrics, advanced security)
- Use LangChain4j for all LLM, RAG, and AI integration tasks
- Use Quarkus Qute for server-side templating instead of a separate API/frontend architecture

### Dependency Versions (Hard Requirements):
```
io.quarkus:quarkus-bom:3.11.1
io.quarkus:quarkus-resteasy-reactive:3.11.1
io.quarkus:quarkus-resteasy-reactive-jackson:3.11.1
io.quarkus:quarkus-rest-client-reactive-jackson:3.11.1
io.quarkus:quarkus-kotlin:3.11.1
io.quarkus:quarkus-arc:3.11.1
io.quarkus:quarkus-resteasy-reactive-qute:3.11.1
org.jetbrains.kotlin:kotlin-stdlib:1.9.23
dev.langchain4j:langchain4j:1.0.0-beta3
dev.langchain4j:langchain4j-open-ai:1.0.0-beta3
org.jboss.logging:jboss-logging:3.5.3.Final
```
Use these exact versions unless a newer, stable release is explicitly approved.

## 2. Project Structure - CRITICAL GUIDELINES
- MAXIMUM 5-6 Kotlin files total for the core implementation
- Single controller for all endpoints - NO separate controllers per feature
- Single service layer for business logic
- Use Qute templates in the `templates/{ControllerName}` directory instead of static HTML
- Use built-in Quarkus features for HTML templating - NO JavaScript complexity
- When in doubt, choose the simpler solution over the "cleaner" architecture

## 3. LangChain4j Integration - MANDATORY RULES
- **ALWAYS CHECK OFFICIAL DOCUMENTATION FIRST**: Before writing any LangChain4j code, consult the [official documentation](https://docs.langchain4j.dev/) for the specific version being used (1.0.0-beta3).
- **VERIFY API SIGNATURES**: Package structures and method signatures in LangChain4j may change between versions. Always verify you're using the correct imports and method signatures.
- **MODEL AGNOSTIC IMPLEMENTATION**: Design the integration to work with any LLM provider supported by LangChain4j, not just a specific provider.
- **UNDERSTAND MESSAGE HANDLING**: Pay special attention to how messages (SystemMessage, UserMessage, AiMessage) are implemented in the specific version being used.
- **RESEARCH COMMON PATTERNS**: Research and understand the common patterns for LangChain4j before implementing, particularly for:
  - Model initialization
  - Message formatting and sending
  - Response handling
- **INVESTIGATE ERRORS**: If encountering build errors, check package imports first, as they're the most common issue with LangChain4j integration.
- Consult the official documentation and examples at https://github.com/langchain4j/langchain4j-examples

## 4. API Protocol - SIMPLIFIED APPROACH

### Backend Endpoints & Data Formats

**UI Endpoint**: `GET /`
- Serves the HTML UI with Qute templating

**Form Processing**: `POST /`
- Accepts form data with a `query` parameter
- Returns HTML response with the AI-generated answer directly rendered in the page

### Error Handling Protocol
- Render error messages directly in the template
- Use server-side validation for input
- Provide clear user feedback for errors

## 5. Code Quality & Configuration
- Don't overcomplicate, avoid unnecessary abstractions
- Write idiomatic, concise, and readable code
- Anticipate future extensibility (RAG, local models, new endpoints)
- Externalize configuration via application.properties or environment variables
- Limit configuration to only what's necessary for the demo

## 6. Error Handling and Logging
- Implement MINIMAL but sufficient error handling for demonstration purposes
- Display errors directly in the template
- Focus on user-facing error messages over complex error infrastructure
- Log errors clearly for debugging but avoid complex logging frameworks
- Log all incoming requests and errors to the console for traceability

## 7. Frontend Requirements
- Use Qute template rendering instead of client-side JavaScript
- Display error messages in the template
- Use clean, minimal styling
- Avoid client-side JavaScript complexity

## 8. Unnecessary Features - DELIBERATELY OMITTED
- **CORS Configuration**: With server-side templating, there are no cross-origin requests, so CORS is completely unnecessary
- **Health Endpoints**: Health endpoints are unnecessary with a simplified server-side rendering architecture
- **Custom Error Mappers**: With direct template rendering, complex JSON error handling is redundant
- **Client-side API Interaction**: Eliminated in favor of simple form submissions and server-side rendering

## 9. Deliverables and Documentation
- README must document: Setup, environment variables, and how to run/test
- Deliverables must include: backend, template files, build files, and README

## 10. Zero-Debugging, Demo-Ready Delivery
- Deliver code that is robust and demo-ready in a single step
- Focus on making it WORK rather than making it architecturally perfect
- The demo should start successfully with a single command (./gradlew quarkusDev)
