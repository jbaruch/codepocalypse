# Running Local Mistral Model with Input Guardrails in Spring AI

This guide shows how to configure a local Mistral model running in Docker and implement input guardrails to restrict queries to airline loyalty topics only, using Spring AI.

## Prerequisites

- Docker Desktop installed and running
- Mistral model (mistral:7B-Q4_K_M) already pulled and running locally
- Ollama or compatible server running on port 11434 (default)

## Part 1: Configure Local Mistral Model

### Option 1: Property-Based Configuration (Recommended)

Update your `application.yml`:

```yaml
spring:
  ai:
    openai:
      api-key: not-needed  # Placeholder required
      base-url: http://localhost:11434/v1
      chat:
        options:
          model: mistral:7B-Q4_K_M
          temperature: 0.7
```

### Option 2: Programmatic Configuration

```kotlin
@Configuration
class LocalMistralConfiguration {
    
    @Bean
    fun localMistralChatModel(): ChatModel {
        val api = OpenAiApi(
            "http://localhost:11434/v1",
            "not-needed"
        )
        
        val options = OpenAiChatOptions.builder()
            .model("mistral:7B-Q4_K_M")
            .temperature(0.7)
            .build()
        
        return OpenAiChatModel(api, options)
    }
}
```

## Part 2: Implement Input Guardrails

Spring AI uses **Advisors** to implement guardrails. Advisors intercept requests before they reach the model, allowing you to validate, modify, or reject them.

### Create a Topic Validation Advisor

```kotlin
@Component
class AirlineLoyaltyGuardrailAdvisor(
    private val validationChatModel: ChatModel
) : CallAdvisor, StreamAdvisor {
    
    private val logger = LoggerFactory.getLogger(AirlineLoyaltyGuardrailAdvisor::class.java)
    
    companion object {
        private const val VALIDATION_SYSTEM_PROMPT = """
            You are a content validator. Your task is to determine if a user's question 
            is related to airline loyalty programs.
            
            Airline loyalty topics include:
            - Frequent flyer programs (Delta SkyMiles, United MileagePlus, etc.)
            - Status tiers and qualification requirements
            - Earning and redeeming miles or points
            - Elite benefits and perks
            - Award flights and upgrades
            - Airline alliances and partnerships
            
            Respond with ONLY "YES" if the question is about airline loyalty programs.
            Respond with ONLY "NO" if it is not.
            
            Do not provide any explanation, just YES or NO.
        """
    }
    
    override fun getName(): String = "AirlineLoyaltyGuardrailAdvisor"
    
    override fun getOrder(): Int = Ordered.HIGHEST_PRECEDENCE // Run first
    
    override fun adviseCall(
        request: ChatClientRequest,
        chain: CallAdvisorChain
    ): ChatClientResponse {
        val userMessage = request.prompt().userMessage.text
        
        if (!isAirlineLoyaltyTopic(userMessage)) {
            logger.warn("Rejected off-topic question: $userMessage")
            throw GuardrailViolationException(
                "I'm specialized in airline loyalty programs. " +
                "Please ask questions about frequent flyer programs, status tiers, " +
                "miles, or airline rewards."
            )
        }
        
        logger.info("Question passed guardrail validation")
        return chain.nextCall(request)
    }
    
    override fun adviseStream(
        request: ChatClientRequest,
        chain: StreamAdvisorChain
    ): Flux<ChatClientResponse> {
        val userMessage = request.prompt().userMessage.text
        
        if (!isAirlineLoyaltyTopic(userMessage)) {
            logger.warn("Rejected off-topic question: $userMessage")
            return Flux.error(
                GuardrailViolationException(
                    "I'm specialized in airline loyalty programs. " +
                    "Please ask questions about frequent flyer programs, status tiers, " +
                    "miles, or airline rewards."
                )
            )
        }
        
        logger.info("Question passed guardrail validation")
        return chain.nextStream(request)
    }
    
    private fun isAirlineLoyaltyTopic(userMessage: String): Boolean {
        val validationPrompt = Prompt(
            listOf(
                SystemMessage(VALIDATION_SYSTEM_PROMPT),
                UserMessage(userMessage)
            )
        )
        
        return try {
            val response = validationChatModel.call(validationPrompt)
            val result = response.result.output.content.trim().uppercase()
            
            logger.debug("Validation result for '$userMessage': $result")
            result == "YES"
        } catch (e: Exception) {
            logger.error("Error during guardrail validation", e)
            // Fail open: allow the question if validation fails
            true
        }
    }
}

// Custom exception for guardrail violations
class GuardrailViolationException(message: String) : RuntimeException(message)
```

### Register the Guardrail Advisor

```kotlin
@Configuration
class ChatClientConfiguration {
    
    @Bean
    fun chatClient(
        chatModel: ChatModel,
        chatMemory: ChatMemory,
        guardrailAdvisor: AirlineLoyaltyGuardrailAdvisor
    ): ChatClient {
        return ChatClient.builder(chatModel)
            .defaultSystem("""
                You are a helpful airline loyalty program assistant specializing in 
                Delta SkyMiles and United MileagePlus programs. Answer questions about 
                status qualification, benefits, earning miles, and redeeming rewards.
            """.trimIndent())
            .defaultAdvisors(
                guardrailAdvisor,  // Input validation - runs first
                MessageChatMemoryAdvisor.builder(chatMemory).build()
            )
            .build()
    }
    
    // Separate chat model for validation (can be same or different)
    @Bean
    @Qualifier("validationChatModel")
    fun validationChatModel(): ChatModel {
        // Use same local model for validation
        val api = OpenAiApi(
            "http://localhost:11434/v1",
            "not-needed"
        )
        
        val options = OpenAiChatOptions.builder()
            .model("mistral:7B-Q4_K_M")
            .temperature(0.0)  // Use 0 for deterministic validation
            .build()
        
        return OpenAiChatModel(api, options)
    }
}
```

### Update Service to Handle Guardrail Violations

```kotlin
@Service
class AiService(private val chatClient: ChatClient) {
    private val logger = LoggerFactory.getLogger(AiService::class.java)
    
    fun chat(userMessage: String, conversationId: String): String {
        return try {
            chatClient.prompt()
                .user(userMessage)
                .advisors { advisor -> 
                    advisor.param(ChatMemory.CONVERSATION_ID, conversationId)
                }
                .call()
                .content()
        } catch (e: GuardrailViolationException) {
            logger.info("Guardrail rejected query: ${e.message}")
            e.message ?: "Your question is outside my area of expertise."
        }
    }
}
```

### Update Controller

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
            val response = aiService.chat(query, conversationId)
            model.addAttribute("response", response)
        } catch (e: Exception) {
            model.addAttribute("error", e.message)
        }
        
        model.addAttribute("query", query)
        model.addAttribute("hasConversation", true)
        return "chat/index"
    }
}
```

## Alternative: Simple Keyword-Based Guardrail

For simpler cases without LLM-based validation:

```kotlin
@Component
class SimpleAirlineGuardrailAdvisor : CallAdvisor, StreamAdvisor {
    
    private val logger = LoggerFactory.getLogger(SimpleAirlineGuardrailAdvisor::class.java)
    
    companion object {
        private val AIRLINE_KEYWORDS = setOf(
            "airline", "flight", "miles", "points", "status", "tier",
            "delta", "skymiles", "united", "mileageplus", "premier", "medallion",
            "frequent", "flyer", "loyalty", "rewards", "elite", "upgrade",
            "pqp", "pqf", "mqm", "mqd", "mqs"
        )
    }
    
    override fun getName(): String = "SimpleAirlineGuardrailAdvisor"
    override fun getOrder(): Int = Ordered.HIGHEST_PRECEDENCE
    
    override fun adviseCall(
        request: ChatClientRequest,
        chain: CallAdvisorChain
    ): ChatClientResponse {
        validateAirlineTopic(request.prompt().userMessage.text)
        return chain.nextCall(request)
    }
    
    override fun adviseStream(
        request: ChatClientRequest,
        chain: StreamAdvisorChain
    ): Flux<ChatClientResponse> {
        validateAirlineTopic(request.prompt().userMessage.text)
        return chain.nextStream(request)
    }
    
    private fun validateAirlineTopic(userMessage: String) {
        val lowerMessage = userMessage.lowercase()
        val hasAirlineKeyword = AIRLINE_KEYWORDS.any { keyword ->
            lowerMessage.contains(keyword)
        }
        
        if (!hasAirlineKeyword) {
            logger.warn("Rejected off-topic question (no airline keywords): $userMessage")
            throw GuardrailViolationException(
                "I'm specialized in airline loyalty programs. " +
                "Please ask about frequent flyer programs, miles, or status tiers."
            )
        }
        
        logger.info("Question contains airline keywords: passed validation")
    }
}
```

## Complete Example with Local Model and Guardrails

```kotlin
// Main Configuration
@Configuration
class AirlineAssistantConfiguration {
    
    @Bean
    fun localMistralChatModel(): ChatModel {
        val api = OpenAiApi(
            "http://localhost:11434/v1",
            "not-needed"
        )
        
        val options = OpenAiChatOptions.builder()
            .model("mistral:7B-Q4_K_M")
            .temperature(0.7)
            .maxTokens(500)
            .build()
        
        return OpenAiChatModel(api, options)
    }
    
    @Bean
    fun chatMemory(): ChatMemory {
        return MessageWindowChatMemory.builder()
            .maxMessages(20)
            .build()
    }
    
    @Bean
    fun chatClient(
        chatModel: ChatModel,
        chatMemory: ChatMemory,
        guardrailAdvisor: AirlineLoyaltyGuardrailAdvisor
    ): ChatClient {
        return ChatClient.builder(chatModel)
            .defaultSystem("""
                You are an airline loyalty program specialist. You help users understand
                Delta SkyMiles and United MileagePlus programs. Be concise and helpful.
            """.trimIndent())
            .defaultAdvisors(
                guardrailAdvisor,
                MessageChatMemoryAdvisor.builder(chatMemory).build()
            )
            .build()
    }
}

// Service
@Service
class AiService(private val chatClient: ChatClient) {
    private val logger = LoggerFactory.getLogger(AiService::class.java)
    
    fun chat(userMessage: String, conversationId: String): ChatResult {
        return try {
            val response = chatClient.prompt()
                .user(userMessage)
                .advisors { advisor -> 
                    advisor.param(ChatMemory.CONVERSATION_ID, conversationId)
                }
                .call()
                .content()
            
            ChatResult(
                response = response,
                passed = true,
                usingLocalModel = true
            )
        } catch (e: GuardrailViolationException) {
            logger.info("Guardrail violation: ${e.message}")
            ChatResult(
                response = e.message ?: "Question rejected",
                passed = false,
                usingLocalModel = true
            )
        }
    }
}

data class ChatResult(
    val response: String,
    val passed: Boolean,
    val usingLocalModel: Boolean
)
```

## Configuration (application.yml)

```yaml
spring:
  ai:
    openai:
      api-key: not-needed
      base-url: http://localhost:11434/v1
      chat:
        options:
          model: mistral:7B-Q4_K_M
          temperature: 0.7
          max-tokens: 500

server:
  port: 8080

logging:
  level:
    com.example.airlineloyalty: DEBUG
```

## Testing Your Implementation

### Start Docker and Ollama

```bash
# Verify Ollama is running
docker ps | grep ollama

# If not running, start it:
docker run -d -p 11434:11434 \
  --name ollama \
  -v ollama:/root/.ollama \
  ollama/ollama

# Verify model is available
curl http://localhost:11434/api/tags
```

### Test the Application

```bash
# Start the application
./gradlew bootRun

# The application should log:
# - "Using local Mistral model at http://localhost:11434/v1"
# - "Guardrail advisor registered"
```

### Test Cases

**Valid Questions (Should Pass)**:
- "What are the requirements for Delta Gold Medallion?"
- "How do I earn United Premier status?"
- "Can I use miles to upgrade my flight?"
- "What benefits do Delta Platinum members get?"

**Invalid Questions (Should Be Rejected)**:
- "What's the weather like today?"
- "Tell me a joke"
- "How do I cook pasta?"
- "What's the capital of France?"

### Example Interactions

**Valid Query**:
```
User: "How do I qualify for Delta Silver Medallion?"
Guardrail: ✓ Passed validation
Response: "To qualify for Delta Silver Medallion status, you need..."
```

**Invalid Query**:
```
User: "What's the best pizza in New York?"
Guardrail: ✗ Failed validation
Response: "I'm specialized in airline loyalty programs. 
          Please ask questions about frequent flyer programs, 
          status tiers, miles, or airline rewards."
```


## Notes

- **Advisor Order**: Use `Ordered.HIGHEST_PRECEDENCE` to ensure guardrails run before other advisors
- **Validation Model**: Can use the same local model for both validation and responses, or configure separate models
- **Temperature**: Set to 0.0 for validation to ensure deterministic results
- **Fail Open vs Fail Closed**: Current implementation fails open (allows query) if validation errors occur. Adjust based on your requirements.
- **Performance**: LLM-based validation adds latency. Consider keyword-based approach for faster responses.
- **Local Model**: Responses may be slower and less accurate than GPT-4, but provide privacy and cost benefits.

## Troubleshooting

### Guardrail Always Rejects/Accepts

```kotlin
// Add debug logging to see validation responses
private fun isAirlineLoyaltyTopic(userMessage: String): Boolean {
    val response = validationChatModel.call(validationPrompt)
    val result = response.result.output.content.trim()
    
    logger.debug("Raw validation response: '$result'")
    
    return result.uppercase() == "YES"
}
```

### Local Model Not Responding

```bash
# Check if Ollama is accessible
curl http://localhost:11434/api/tags

# Check model is loaded
curl http://localhost:11434/api/show -d '{"name":"mistral:7B-Q4_K_M"}'

# Test direct inference
curl http://localhost:11434/api/generate -d '{
  "model": "mistral:7B-Q4_K_M",
  "prompt": "Hello"
}'
```

### Validation Takes Too Long

Consider using the keyword-based approach instead of LLM validation:

```kotlin
@Bean
fun guardrailAdvisor(): AirlineLoyaltyGuardrailAdvisor {
    // Use simple keyword-based validation
    return SimpleAirlineGuardrailAdvisor()
}
```

## Advanced: Output Guardrails

You can also implement output guardrails by checking responses before returning them:

```kotlin
@Component
class OutputGuardrailAdvisor : CallAdvisor {
    
    override fun adviseCall(
        request: ChatClientRequest,
        chain: CallAdvisorChain
    ): ChatClientResponse {
        val response = chain.nextCall(request)
        
        // Check response doesn't contain inappropriate content
        val content = response.chatResponse.result.output.content
        if (containsInappropriateContent(content)) {
            throw GuardrailViolationException(
                "Response contained inappropriate content"
            )
        }
        
        return response
    }
    
    override fun getOrder(): Int = Ordered.LOWEST_PRECEDENCE // Run last
}
```

This implementation provides robust input guardrails while maintaining the simplicity and demonstration focus of your airline loyalty assistant application!
