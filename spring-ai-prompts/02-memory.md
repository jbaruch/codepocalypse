# Airline Loyalty Assistant Enhancement Task - Conversation Memory

Enhance the existing Airline Loyalty Assistant web application by implementing conversation memory to provide a more personalized and contextual user experience using Spring AI.

## Enhancement Overview

The current application provides answers about airline loyalty programs using an LLM through Spring AI, but treats each interaction as isolated. This enhancement will:

1. Add the ability to maintain conversation context across multiple interactions
2. Enable the assistant to reference prior messages in the conversation
3. Allow for more natural, flowing conversations with the assistant

## Technical Requirements

- Continue following **ALL** specifications in the `.junie/guidelines.md` file
- Use Spring AI's conversation memory capabilities, specifically `MessageWindowChatMemory` with max 20 messages
- Implement `MessageChatMemoryAdvisor` to integrate memory with the `ChatClient`
- Implement a lightweight approach aligned with the project's minimalist philosophy
- Maintain the existing simplified architecture and UI approach

## Implementation Guidelines

1. **Memory Implementation**:
   - Create a **single application-wide `ChatMemory` bean** using `MessageWindowChatMemory.builder().maxMessages(20).build()`
   - Configure the `ChatClient` with `MessageChatMemoryAdvisor` in the default advisors chain
   - Follow the pattern from the official Spring AI documentation for `MessageChatMemoryAdvisor`
   - Store conversation IDs in HTTP session to track user conversations
   - Example configuration:
     ```kotlin
     @Bean
     fun chatMemory(): ChatMemory {
         return MessageWindowChatMemory.builder()
             .maxMessages(20)
             .build()
     }
     
     @Bean
     fun chatClient(chatModel: ChatModel, chatMemory: ChatMemory): ChatClient {
         return ChatClient.builder(chatModel)
             .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
             .build()
     }
     ```

2. **Service Layer Updates**:
   - Update the AI service to pass a `conversationId` with each chat request
   - Use HTTP session to maintain conversation IDs across requests
   - Pass conversation ID to the advisor using `.advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))`
   - Example:
     ```kotlin
     fun chat(userMessage: String, conversationId: String): String {
         return chatClient.prompt()
             .user(userMessage)
             .advisors { a -> a.param(ChatMemory.CONVERSATION_ID, conversationId) }
             .call()
             .content()
     }
     ```

3. **Enhanced System Message**:
   - Update the system message configuration to explicitly instruct the AI to remember user details
   - Include specific instructions to remember names, preferences, and other personal information shared during the conversation
   - Configure this via the `ChatClient` builder or in application.yml

4. **Session Management**:
   - Use Spring's `HttpSession` to store and retrieve conversation IDs
   - Generate a unique conversation ID when a new session starts
   - Pass the conversation ID from controller to service layer

5. **UI Enhancements**:
   - Add a visual indicator when a conversation is active
   - Add a "New Conversation" button to reset the conversation context
   - Modify prompts to encourage follow-up questions when in an active conversation

6. **Response Improvement**:
   - Enable more natural conversation flows and contextual responses
   - Support references to past interactions without requiring the user to repeat information

## User Experience

- The UI should remain simple with minimal changes except for conversation indicators
- Users should be able to ask follow-up questions without restating context
- The assistant should remember user preferences and information shared during the conversation
- Users can start a new conversation to reset the context

## Development Approach

- Maintain the same simplified architecture as the original implementation
- Keep the core principles of minimalism and demo-readiness
- Prioritize working functionality over architectural complexity
- Ensure the application continues to start with a single command (`./gradlew bootRun`)
- Use in-memory storage for the conversation memory (default `InMemoryChatMemoryRepository`)

## Code Examples

### Configuration Class
```kotlin
@Configuration
class ChatConfiguration {
    
    @Bean
    fun chatMemory(): ChatMemory {
        return MessageWindowChatMemory.builder()
            .maxMessages(20)
            .build()
    }
    
    @Bean
    fun chatClient(
        chatModel: ChatModel,
        chatMemory: ChatMemory
    ): ChatClient {
        return ChatClient.builder(chatModel)
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory).build()
            )
            .build()
    }
}
```

### Updated Service
```kotlin
@Service
class AiService(private val chatClient: ChatClient) {
    
    fun chat(userMessage: String, conversationId: String): String {
        return chatClient.prompt()
            .user(userMessage)
            .advisors { advisor -> 
                advisor.param(ChatMemory.CONVERSATION_ID, conversationId) 
            }
            .call()
            .content()
    }
}
```

### Updated Controller
```kotlin
@Controller
class ChatController(private val aiService: AiService) {
    
    @GetMapping("/")
    fun index(session: HttpSession, model: Model): String {
        val conversationId = session.getAttribute("conversationId") as? String
            ?: UUID.randomUUID().toString().also { 
                session.setAttribute("conversationId", it) 
            }
        model.addAttribute("hasConversation", true)
        return "chat/index"
    }
    
    @PostMapping("/")
    fun chat(
        @RequestParam query: String,
        session: HttpSession,
        model: Model
    ): String {
        val conversationId = session.getAttribute("conversationId") as? String
            ?: UUID.randomUUID().toString().also { 
                session.setAttribute("conversationId", it) 
            }
        
        try {
            val response = aiService.chat(query, conversationId)
            model.addAttribute("response", response)
        } catch (e: Exception) {
            model.addAttribute("error", e.message)
        }
        model.addAttribute("query", query)
        model.addAttribute("hasConversation", true)
        return "chat/index"
    }
    
    @PostMapping("/reset")
    fun reset(session: HttpSession): String {
        session.setAttribute("conversationId", UUID.randomUUID().toString())
        return "redirect:/"
    }
}
```

## Testing Considerations

- Test with multi-turn conversations to verify context retention
- Try introducing yourself with "Hello, my name is [name]" and then ask "What's my name?"
- Verify that the assistant correctly references previous parts of the conversation
- Test the "New Conversation" button to ensure it properly resets context
- Ensure the experience feels natural and continuous to users
- Test that different browser sessions maintain separate conversations