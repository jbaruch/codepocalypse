# «Golden Prompts» to start with 

## Requirements

Assuming you have a detailed set of requirements for your application.
You can use tools like ChatGPT, Claude Desktop or Grok to brainstorm and prioritize requirements.
You even can reverse-engineer existing application by using prompt like this:

```
Provide step-by-step guidelines for implementing the chatbot -  Airline Loyality Assistant - based on this project and write the result to docs/requirements.md file.
DO NOT use code snippets in the requirements doc and follow PRD format.
```

## Prompt 1

```
Analyze the `docs/requirements.md` and create a detailed plan for implementing this project. 
Write the plan to docs/plan.md file
```

## Prompt 2

```
Create a detailed enumerated task list according to the suggested enhancements plan in docs/plan.md. 
Task items should have a placeholder [ ] for marking as done [x] upon task completion. 
Write the task list to docs/tasks.md file
```

## Prompt 3

```
Proceed implementing the application according to the tasks listed in docs/tasks.md. 
Start with Phase 1. 
Mark tasks as done [x] upon completion
```

## References

- Baruch Sadogursky - [Prompt Driven Development](https://speaking.jbaru.ch/yaBltt/prompt-driven-development-aligning-ideas-tests-and-code) 
- Anto Arhipov - [Go To Gym](https://github.com/antonarhipov/gotogym) 