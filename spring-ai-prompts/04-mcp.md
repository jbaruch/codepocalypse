# Airline Loyalty Assistant Enhancement Task - MCP Server Implementation

Enhance the Airline Loyalty Assistant web application by implementing a Model Context Protocol (MCP) server using Spring AI, focusing on:

1. Create a Spring AI MCP server to provide airline qualification information as tools/resources
2. Expose information about Delta SkyMiles and United MileagePlus qualification requirements
3. Integrate the MCP server with the existing Spring AI chat assistant
4. Maintain the existing simplified architecture and UI approach

## Technical Requirements

- Continue following **ALL** specifications in the `.junie/guidelines.md` file
- Use Context7 MCP to look up Spring AI MCP documentation before implementing
- Use ONLY Spring AI's built-in MCP server capabilities
- Use Kotlin for all implementation code
- Keep the implementation simple and demo-focused
- Maintain the existing UI and form submission approach

## MCP Server Implementation Strategy

Create a standalone MCP server application that the chat assistant connects to as a client. 
This better demonstrates the MCP protocol but requires more setup.

## Implementation Guidelines

### 1. MCP Server Setup

**Add Required Dependencies** to `build.gradle.kts`:

```kotlin
dependencies {
    // Existing dependencies...
    
    // MCP Server support (WebMVC for synchronous operations)
    implementation("org.springframework.ai:spring-ai-starter-mcp-server-webmvc")
    
    // Or for reactive applications:
    // implementation("org.springframework.ai:spring-ai-starter-mcp-server-webflux")
}
```

**Configure MCP Server** in `application.yml`:
```yaml
spring:
  ai:
    mcp:
      server:
        enabled: true
        name: airline-qualification-server
        version: 1.0.0
        type: SYNC  # or ASYNC for reactive
        transport: WEBMVC  # or WEBFLUX
        sse-message-endpoint: /mcp/messages
        instructions: "Provides airline loyalty program qualification information for Delta and United"
        capabilities:
          tool: true
          resource: true
```

### 2. Create Airline Qualification Tools

Define tools that can query airline qualification information:

```kotlin
@Service
class AirlineQualificationTools(
    private val restTemplate: RestTemplate
) {
    private val logger = LoggerFactory.getLogger(AirlineQualificationTools::class.java)
    
    // In-memory cache of qualification data
    private val qualificationData = mutableMapOf<String, String>()
    
    @PostConstruct
    fun loadQualificationData() {
        // Load Delta qualification info
        qualificationData["delta"] = loadAirlineInfo(
            "https://www.delta.com/us/en/skymiles/medallion-program/how-to-qualify",
            "Delta SkyMiles Medallion"
        )
        
        // Load United qualification info
        qualificationData["united"] = loadAirlineInfo(
            "https://www.united.com/en/us/fly/mileageplus/premier/qualify.html",
            "United MileagePlus Premier"
        )
    }
    
    private fun loadAirlineInfo(url: String, programName: String): String {
        return try {
            logger.info("Loading qualification info from: $url")
            val html = restTemplate.getForObject(url, String::class.java)
            val cleanedText = cleanHtml(html ?: "")
            
            // Store condensed version suitable for tool responses
            "$programName Qualification Requirements:\n\n$cleanedText"
        } catch (e: Exception) {
            logger.error("Failed to load $programName info", e)
            "Unable to load qualification information for $programName"
        }
    }
    
    private fun cleanHtml(html: String): String {
        val doc = Jsoup.parse(html)
        doc.select("script, style, nav, header, footer, iframe").remove()
        return doc.body().text()
            .replace("\\s+".toRegex(), " ")
            .trim()
            .take(4000) // Limit length for tool responses
    }
    
    @Tool(description = "Get Delta SkyMiles Medallion status qualification requirements including MQMs, MQDs, and MQSs needed for Silver, Gold, Platinum, and Diamond status")
    fun getDeltaQualification(): String {
        return qualificationData["delta"] 
            ?: "Delta qualification information is not available"
    }
    
    @Tool(description = "Get United MileagePlus Premier status qualification requirements including PQPs, PQFs, and alternative paths for Silver, Gold, Platinum, and 1K status")
    fun getUnitedQualification(): String {
        return qualificationData["united"]
            ?: "United qualification information is not available"
    }
    
    @Tool(description = "Compare qualification requirements between Delta and United loyalty programs")
    fun compareAirlinePrograms(): String {
        val delta = qualificationData["delta"]
        val united = qualificationData["united"]
        
        return """
            AIRLINE LOYALTY PROGRAM COMPARISON
            
            Delta SkyMiles Medallion:
            ${delta?.take(1000) ?: "Not available"}
            
            ---
            
            United MileagePlus Premier:
            ${united?.take(1000) ?: "Not available"}
        """.trimIndent()
    }
}
```

### 3. Register Tools with MCP Server

Create a configuration to expose tools:

```kotlin
@Configuration
class McpServerConfiguration {
    
    @Bean
    fun airlineToolCallbackProvider(
        airlineTools: AirlineQualificationTools
    ): ToolCallbackProvider {
        return MethodToolCallbackProvider.builder()
            .toolObjects(airlineTools)
            .build()
    }
}
```

### 4. Alternative: Use MCP Resources Instead of Tools

If you prefer to expose data as resources rather than tools:

```kotlin
@Configuration
class McpResourceConfiguration {
    
    @Bean
    fun airlineResources(
        airlineTools: AirlineQualificationTools
    ): List<McpServerFeatures.SyncResourceSpecification> {
        val objectMapper = ObjectMapper()
        
        // Delta qualification resource
        val deltaResource = McpSchema.Resource(
            URI.create("airline://delta/qualification"),
            "Delta SkyMiles Medallion Qualification",
            "Qualification requirements for Delta SkyMiles status tiers",
            "text/plain"
        )
        
        val deltaSpec = McpServerFeatures.SyncResourceSpecification(deltaResource) { exchange, request ->
            val content = airlineTools.getDeltaQualification()
            McpSchema.ReadResourceResult(
                listOf(
                    McpSchema.TextResourceContents(
                        request.uri(),
                        "text/plain",
                        content
                    )
                )
            )
        }
        
        // United qualification resource
        val unitedResource = McpSchema.Resource(
            URI.create("airline://united/qualification"),
            "United MileagePlus Premier Qualification",
            "Qualification requirements for United MileagePlus status tiers",
            "text/plain"
        )
        
        val unitedSpec = McpServerFeatures.SyncResourceSpecification(unitedResource) { exchange, request ->
            val content = airlineTools.getUnitedQualification()
            McpSchema.ReadResourceResult(
                listOf(
                    McpSchema.TextResourceContents(
                        request.uri(),
                        "text/plain",
                        content
                    )
                )
            )
        }
        
        return listOf(deltaSpec, unitedSpec)
    }
}
```

### 5. Integrate MCP Tools with Chat Assistant

Update the chat configuration to include MCP tools:

```kotlin
@Configuration
class ChatConfiguration {
    
    @Bean
    fun chatClient(
        chatModel: ChatModel,
        chatMemory: ChatMemory,
        airlineToolCallbackProvider: ToolCallbackProvider
    ): ChatClient {
        return ChatClient.builder(chatModel)
            .defaultSystem("""
                You are a helpful airline loyalty program assistant specializing in Delta SkyMiles 
                and United MileagePlus programs.
                
                You have access to tools that provide current qualification requirements from official 
                airline sources. Use these tools when answering questions about:
                - Status qualification requirements (MQMs, MQDs, PQPs, PQFs, etc.)
                - Elite tier benefits
                - How to earn or maintain status
                
                Always use the tools to get accurate, up-to-date information rather than relying 
                on potentially outdated knowledge.
            """.trimIndent())
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory).build()
            )
            .defaultTools(airlineToolCallbackProvider.toolCallbacks.toList())
            .build()
    }
}
```

### 6. Enhanced Service Layer

Update the service to leverage MCP tools:

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
    
    fun chatWithToolUsage(userMessage: String, conversationId: String): Map<String, Any> {
        val response = chatClient.prompt()
            .user(userMessage)
            .advisors { advisor -> 
                advisor.param(ChatMemory.CONVERSATION_ID, conversationId)
            }
            .call()
            .chatResponse()
        
        // Check if tools were used
        val toolCalls = response.results
            .flatMap { it.metadata?.get("finishReason")?.toString()?.let { listOf(it) } ?: emptyList() }
        
        return mapOf(
            "response" to response.result.output.content,
            "toolsUsed" to toolCalls.isNotEmpty()
        )
    }
}
```

### 7. UI Enhancement (Optional)

Update the controller to indicate when MCP tools were used:

```kotlin
@Controller
class ChatController(private val aiService: AiService) {
    
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
            val result = aiService.chatWithToolUsage(query, conversationId)
            
            model.addAttribute("response", result["response"])
            model.addAttribute("toolsUsed", result["toolsUsed"])
        } catch (e: Exception) {
            model.addAttribute("error", e.message)
        }
        
        model.addAttribute("query", query)
        model.addAttribute("hasConversation", true)
        return "chat/index"
    }
}
```

Update template to show tool usage:

```html
<div th:if="${response}" class="response-section">
    <h3>Response:</h3>
    <p th:text="${response}"></p>
    
    <div th:if="${toolsUsed}" class="tool-indicator">
        <small>✓ Response generated using real-time airline qualification data</small>
    </div>
</div>
```

## Key Differences from LangChain4j/Quarkus

| Aspect | LangChain4j/Quarkus | Spring AI |
|--------|---------------------|-----------|
| **Server Setup** | Custom MCP implementation | Built-in MCP server starters |
| **Tool Definition** | Interface-based with annotations | `@Tool` annotations or `ToolCallback` interface |
| **Transport** | STDIO or custom | STDIO, WebMVC (SSE), or WebFlux (SSE) |
| **Configuration** | application.properties | application.yml or application.properties |
| **Integration** | CDI-based | Spring dependency injection |
| **Resources** | Not standard in LangChain4j | First-class `Resource` support in MCP |

## Configuration (application.yml)

```yaml
spring:
  ai:
    openai:
      api-key: ${SPRING_AI_OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4
          temperature: 0.7
    
    mcp:
      server:
        enabled: true
        name: airline-qualification-server
        version: 1.0.0
        type: SYNC
        transport: WEBMVC
        sse-message-endpoint: /mcp/messages
        instructions: "Provides airline qualification information"
        capabilities:
          tool: true
          resource: true
          prompt: false
          completion: false

server:
  port: 8080
```

## Testing Your Implementation

1. **Start the Application**:
   ```bash
   ./gradlew bootRun
   ```

2. **Verify MCP Server**:
   - Check logs for: "MCP server started"
   - Tools should be automatically registered

3. **Test Tool Calls**:
   Ask questions that should trigger tool usage:
   - "What are the requirements for Delta Gold Medallion?"
   - "How do I qualify for United Premier Silver?"
   - "Compare Delta and United qualification requirements"

4. **Verify Tool Execution**:
   - Check logs for tool call indicators
   - Verify responses include accurate qualification information
   - Confirm UI shows tool usage indicator

5. **Test MCP Endpoint** (if using WebMVC):
   - MCP SSE endpoint should be available at `/mcp/messages`
   - Can be tested with MCP client tools

## Common Pitfalls to Avoid

1. **Tool Descriptions**: Make tool descriptions clear and specific. The AI model uses these to decide when to call tools.

2. **Data Loading**: Load airline qualification data during startup (`@PostConstruct`) to avoid delays during chat interactions.

3. **Content Size**: MCP tools should return concise information. Large text blocks may exceed token limits.

4. **Error Handling**: Always handle web scraping errors gracefully. Have fallback data or clear error messages.

5. **Transport Configuration**: WebMVC transport requires `spring-boot-starter-web`. WebFlux requires `spring-boot-starter-webflux`. Don't mix them.

6. **Tool Registration**: Tools must be registered as Spring beans and exposed via `ToolCallbackProvider` for automatic discovery.

7. **HTML Cleaning**: Use Jsoup to clean HTML properly. Raw HTML in tool responses will confuse the AI model.

## Example Test Queries

- "What are the requirements to earn Delta Silver Medallion status?"
- "How many PQPs do I need for United Gold?"
- "Compare the qualification requirements between Delta and United"
- "What's the difference between MQMs and PQFs?"
- "Can I use a credit card to help qualify for status?"

## Performance Considerations

- Load airline data once at startup, not on every request
- Cache cleaned HTML content in memory
- Keep tool response sizes under 2000-4000 characters
- Use SYNC mode for simplicity unless high concurrency is needed
- Consider implementing a refresh mechanism for data updates

## Future Enhancements

- Add more airlines (American, Southwest, etc.)
- Implement data refresh on a schedule
- Add tools for benefit comparisons
- Expose MCP server for external clients
- Add prompt templates as MCP prompts
- Implement resource subscriptions for data updates

This implementation provides a working MCP server that enhances your airline loyalty assistant with real-time qualification data from official airline sources, while maintaining the simple, demonstration-focused architecture.