# Airline Loyalty Assistant Implementation Task

Implement a minimal, production-ready Airline Loyalty Assistant web application that helps users with questions about airline loyalty programs, miles, rewards, and travel benefits.

## Core Requirements

- Follow **ALL** specifications in the `.junie/guidelines.md` file at the root of the repository
- The `.junie/guidelines.md` file is the authoritative source for all implementation details
- Any implementation decisions should defer to `.junie/guidelines.md` when there is a conflict or ambiguity

## Application Overview

This is a simple web application where:

1. Users can enter questions about airline loyalty programs in a web form
2. The application processes these questions using an LLM through LangChain4j
3. Responses are displayed directly in the web UI using Quarkus Qute templates

## Key Features

- Question answering about airline loyalty programs
- Server-side templating for a simple, lightweight UI
- OpenAI integration through LangChain4j
- Error handling for invalid inputs and API failures

## Development Guidelines

- **ALWAYS** check the `.junie/guidelines.md` file for specific technical requirements
- Prioritize simplicity and working code over architectural complexity
- Ensure a zero-debugging, demo-ready delivery that starts with `./gradlew quarkusDev`
- Maintain proper error handling and logging for a good user experience
