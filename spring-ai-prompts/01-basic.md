# Airline Loyalty Assistant Implementation Task

Implement a minimal, production-ready Airline Loyalty Assistant web application that helps users with questions about airline loyalty programs, miles, rewards, and travel benefits.

## Core Requirements

- Follow **ALL** specifications in the `.junie/guidelines.md` file at the root of the repository
- The `.junie/guidelines.md` file is the authoritative source for all implementation details
- Any implementation decisions should defer to `.junie/guidelines.md` when there is a conflict or ambiguity

## Application Overview

This is a simple web application where:
1. Users can enter questions about airline loyalty programs in a web form
2. The application processes these questions using an LLM through Spring AI
3. Responses are displayed directly in the web UI using Thymeleaf templates

## Key Features

- Question answering about airline loyalty programs
- Server-side templating for a simple, lightweight UI
- OpenAI integration through Spring AI
- Error handling for invalid inputs and API failures

## Development Guidelines

- **ALWAYS** check the `.junie/guidelines.md` file for specific technical requirements
- Prioritize simplicity and working code over architectural complexity
- Ensure a zero-debugging, demo-ready delivery that starts with `./gradlew bootRun`
- Maintain proper error handling and logging for a good user experience

## Implementation Checklist

### 1. Project Setup
- [ ] Initialize Spring Boot project with Gradle + Kotlin
- [ ] Add Spring AI 1.0.2 dependencies to build.gradle.kts
- [ ] Add Thymeleaf dependency for templating
- [ ] Configure application.yml with OpenAI API key placeholder

### 2. Configuration
- [ ] Create AI configuration class with @Bean for ChatModel
- [ ] Set up environment variable for SPRING_AI_OPENAI_API_KEY
- [ ] Configure appropriate model and temperature settings

### 3. Service Layer
- [ ] Create AiService with ChatModel injection
- [ ] Implement chat method that processes user queries
- [ ] Add proper error handling for API failures

### 4. Controller
- [ ] Create ChatController with GET and POST mappings
- [ ] Handle form submission with query parameter
- [ ] Pass response/error data to Thymeleaf model

### 5. Templates
- [ ] Create Thymeleaf template for chat interface
- [ ] Add form for user input
- [ ] Display AI response or error messages
- [ ] Style with minimal CSS for professional appearance

### 6. Testing & Documentation
- [ ] Test with `./gradlew bootRun`
- [ ] Verify hot reload with Spring Boot DevTools
- [ ] Create README.md with setup instructions
- [ ] Document required environment variables

## Success Criteria

✅ Application starts successfully with `./gradlew bootRun`  
✅ Users can submit questions via web form  
✅ AI responses are displayed in the same page  
✅ Errors are handled gracefully with user-friendly messages  
✅ Code follows Kotlin best practices and Spring conventions  
✅ README.md provides clear setup and usage instructions  
✅ No build warnings or errors  
✅ Application is demo-ready without additional configuration (except API key)