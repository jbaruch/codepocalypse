# Airline Loyalty Assistant Enhancement Task - Memory

Enhance the existing Airline Loyalty Assistant web application by implementing conversation memory to provide a more personalized and contextual user experience.

## Enhancement Overview

The current application provides answers about airline loyalty programs using an LLM through LangChain4j, but treats each interaction as isolated. This enhancement will:

1. Add the ability to maintain conversation context across multiple interactions
2. Enable the assistant to reference prior messages in the conversation
3. Allow for more natural, flowing conversations with the assistant

## Technical Requirements

- Continue following **ALL** specifications in the `.windsurfrules` file
- Use LangChain4j's conversation memory capabilities, specifically `MessageWindowChatMemory.withMaxMessages(20)`
- Implement a lightweight approach aligned with the project's minimalist philosophy
- Maintain the existing simplified architecture and UI approach

## Implementation Guidelines

1. **Memory Implementation**:
   - Create a **single application-wide conversation memory instance** using `MessageWindowChatMemory.withMaxMessages(20)`
   - Follow the pattern from the official example: [ServiceWithMemoryExample.java](https://github.com/langchain4j/langchain4j-examples/blob/main/other-examples/src/main/java/ServiceWithMemoryExample.java)
   - Store the memory as a private field in the `AssistantService` class
   - Build a single AI assistant instance that uses this memory

2. **Enhanced System Message**:
   - Update the `AirlineAssistantAI` interface's system message to explicitly instruct the AI to remember user details
   - Include specific instructions to remember names, preferences, and other personal information shared during the conversation

3. **UI Enhancements**:
   - Add a visual indicator when a conversation is active
   - Modify prompts to encourage follow-up questions when in an active conversation

4. **Response Improvement**:
   - Enable more natural conversation flows and contextual responses
   - Support references to past interactions without requiring the user to repeat information

## User Experience

- The UI should remain simple and unchanged from the user's perspective except for conversation indicators
- Users should be able to ask follow-up questions without restating context
- The assistant should remember user preferences and information shared during the conversation

## Development Approach

- Maintain the same simplified architecture as the original implementation
- Keep the core principles of minimalism and demo-readiness
- Prioritize working functionality over architectural complexity
- Ensure the application continues to start with a single command (`./gradlew quarkusDev`)
- Use in-memory storage for the conversation memory

## Testing Considerations

- Test with multi-turn conversations to verify context retention
- Try introducing yourself with "Hello, my name is [name]" and then ask "What's my name?"
- Verify that the assistant correctly references previous parts of the conversation
- Ensure the experience feels natural and continuous to users

This enhancement should significantly improve the conversational quality of the assistant while maintaining the lightweight, demonstration-focused nature of the application.
