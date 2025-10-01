# Airline Loyalty Assistant Enhancement Task - In-Memory RAG with Web Content

Enhance the Airline Loyalty Assistant web application by implementing a robust Retrieval Augmented Generation (RAG) system using Spring AI, focusing on:

1. Retrieve and process information from Delta and United loyalty program qualification pages:
   - Delta: https://www.delta.com/us/en/skymiles/medallion-program/how-to-qualify
   - United: https://www.united.com/en/us/fly/mileageplus/premier/qualify.html
2. Use Spring AI's `SimpleVectorStore` for in-memory document storage and efficient information retrieval
3. Load and process web content asynchronously in parallel for improved performance
4. Enhance the quality and accuracy of responses with grounded information from official airline sources

## Technical Requirements

- Continue following **ALL** specifications in the `.junie/guidelines.md` file
- Use ONLY Spring AI's built-in document handling and RAG capabilities
- NEVER implement custom retrieval algorithms or similarity calculations
- Use Spring's `RestTemplate` or `WebClient` for fetching web content
- Clean and extract text content appropriately from HTML
- Maintain the existing simplified architecture and UI approach

## Implementation Guidelines

### 1. Web Content Loading

- Fetch HTML content from airline websites using Spring's `RestTemplate` or `WebClient`
- Extract and clean text content from HTML (remove scripts, styles, navigation)
- Create `Document` objects with proper source metadata for attribution
- DO NOT implement custom similarity search algorithms
- Consider using Jsoup for HTML parsing and cleaning (Spring AI supports `JsoupDocumentReader`)

Example approach:
```kotlin
@Service
class WebContentLoader(
    private val restTemplate: RestTemplate
) {
    private val logger = LoggerFactory.getLogger(WebContentLoader::class.java)
    
    fun loadFromUrl(url: String, title: String): Document? {
        return try {
            val htmlContent = restTemplate.getForObject(url, String::class.java)
            
            // Clean HTML - remove scripts, styles, nav elements
            val cleanedText = cleanHtml(htmlContent)
            
            Document(
                cleanedText,
                mapOf(
                    "source" to url,
                    "title" to title,
                    "type" to "web_page",
                    "loaded_at" to LocalDateTime.now().toString()
                )
            )
        } catch (e: Exception) {
            logger.error("Failed to load content from $url", e)
            null
        }
    }
    
    private fun cleanHtml(html: String?): String {
        if (html.isNullOrBlank()) return ""
        
        // Use Jsoup to parse and extract text
        val doc = Jsoup.parse(html)
        
        // Remove unwanted elements
        doc.select("script, style, nav, header, footer, iframe").remove()
        
        // Get text content
        return doc.body().text()
    }
}
```

**Alternative: Use Spring AI's JsoupDocumentReader**

Add dependency:
```kotlin
implementation("org.springframework.ai:spring-ai-jsoup-document-reader")
```

Then use:
```kotlin
@Service
class WebContentLoader {
    
    fun loadFromUrl(url: String, title: String): List<Document> {
        val resource = UrlResource(url)
        
        val config = JsoupDocumentReaderConfig.builder()
            .selector("main, article, .content") // Target main content areas
            .charset("UTF-8")
            .build()
        
        val reader = JsoupDocumentReader(resource, config)
        val documents = reader.read()
        
        // Add custom metadata
        documents.forEach { doc ->
            doc.metadata["title"] = title
            doc.metadata["source"] = url
        }
        
        return documents
    }
}
```

### 2. Knowledge Base Construction

- Use Spring AI's `TokenTextSplitter` for text chunking
- Use Spring AI's embedding models (`OpenAiEmbeddingModel` or `TransformersEmbeddingModel`) for vector encoding
- Use `SimpleVectorStore` for in-memory vector storage
- NEVER create custom embedding storage or similarity search algorithms

Configuration:
```kotlin
@Configuration
class RagConfiguration {
    
    @Bean
    fun vectorStore(embeddingModel: EmbeddingModel): VectorStore {
        return SimpleVectorStore(embeddingModel)
    }
    
    @Bean
    fun textSplitter(): TokenTextSplitter {
        return TokenTextSplitter(
            defaultChunkSize = 800,
            minChunkSizeChars = 350,
            minChunkLengthToEmbed = 5,
            maxNumChunks = 10000,
            keepSeparator = true
        )
    }
    
    @Bean
    fun restTemplate(): RestTemplate {
        return RestTemplateBuilder()
            .setConnectTimeout(Duration.ofSeconds(10))
            .setReadTimeout(Duration.ofSeconds(30))
            .build()
    }
}
```

### 3. RAG Implementation

- Use Spring AI's `QuestionAnswerAdvisor` or `RetrievalAugmentationAdvisor` for RAG
- Configure the advisor with your vector store
- Use the embedding store's built-in retrieval methods for semantic search
- DO NOT implement custom similarity calculations or ranking algorithms
- Integrate with the existing `ChatClient` using advisors

Integration:
```kotlin
@Bean
fun chatClient(
    chatModel: ChatModel,
    vectorStore: VectorStore,
    chatMemory: ChatMemory
): ChatClient {
    return ChatClient.builder(chatModel)
        .defaultAdvisors(
            MessageChatMemoryAdvisor.builder(chatMemory).build(),
            QuestionAnswerAdvisor.builder(vectorStore)
                .searchRequest(
                    SearchRequest.builder()
                        .topK(5)
                        .similarityThreshold(0.7)
                        .build()
                )
                .build()
        )
        .build()
}
```

### 4. Source Attribution

- Include source URLs in document metadata when creating documents
- Track and display source information for retrieved documents
- Format source attributions clearly in the UI
- Access metadata from retrieved documents to show sources

## Critical Implementation Details for Spring AI 1.0.2

### 1. Required Dependencies

Add to `build.gradle.kts`:
```kotlin
dependencies {
    // Core Spring AI
    implementation("org.springframework.ai:spring-ai-openai-spring-boot-starter")
    
    // RAG support
    implementation("org.springframework.ai:spring-ai-advisors-vector-store")
    
    // For HTML parsing (optional but recommended)
    implementation("org.jsoup:jsoup:1.17.2")
    
    // Alternative: Use Spring AI's Jsoup reader
    // implementation("org.springframework.ai:spring-ai-jsoup-document-reader")
    
    // HTTP client
    implementation("org.springframework.boot:spring-boot-starter-web")
}
```

### 2. Complete Document Loading Service

```kotlin
@Service
class AirlineDocumentLoaderService(
    private val restTemplate: RestTemplate,
    private val textSplitter: TokenTextSplitter,
    private val vectorStore: VectorStore
) {
    private val logger = LoggerFactory.getLogger(AirlineDocumentLoaderService::class.java)
    
    companion object {
        private val AIRLINE_URLS = listOf(
            "https://www.delta.com/us/en/skymiles/medallion-program/how-to-qualify" to "Delta SkyMiles Medallion Qualification",
            "https://www.united.com/en/us/fly/mileageplus/premier/qualify.html" to "United MileagePlus Premier Qualification"
        )
    }
    
    @PostConstruct
    fun loadAirlineDocuments() {
        logger.info("Starting to load airline documents from web...")
        
        // Load documents in parallel
        val documents = AIRLINE_URLS.parallelStream()
            .map { (url, title) -> loadDocument(url, title) }
            .filter { it != null }
            .flatMap { it!!.stream() }
            .toList()
        
        if (documents.isEmpty()) {
            logger.warn("No documents were successfully loaded!")
            return
        }
        
        logger.info("Loaded ${documents.size} documents, splitting into chunks...")
        
        // Split documents into chunks
        val splitDocuments = textSplitter.apply(documents)
        
        logger.info("Split into ${splitDocuments.size} chunks, adding to vector store...")
        
        // Add to vector store (automatically embeds)
        vectorStore.add(splitDocuments)
        
        logger.info("Successfully loaded and embedded ${splitDocuments.size} document chunks from ${AIRLINE_URLS.size} airline pages")
    }
    
    private fun loadDocument(url: String, title: String): List<Document>? {
        return try {
            logger.info("Fetching content from: $url")
            
            val htmlContent = restTemplate.getForObject(url, String::class.java)
            
            if (htmlContent.isNullOrBlank()) {
                logger.warn("Empty content received from $url")
                return null
            }
            
            // Clean HTML content
            val cleanedText = cleanHtml(htmlContent)
            
            if (cleanedText.isBlank()) {
                logger.warn("No text content extracted from $url")
                return null
            }
            
            logger.info("Successfully extracted ${cleanedText.length} characters from $url")
            
            listOf(
                Document(
                    cleanedText,
                    mapOf(
                        "source" to url,
                        "title" to title,
                        "type" to "airline_qualification_page",
                        "loaded_at" to LocalDateTime.now().toString()
                    )
                )
            )
        } catch (e: Exception) {
            logger.error("Failed to load content from $url: ${e.message}", e)
            null
        }
    }
    
    private fun cleanHtml(html: String): String {
        // Parse HTML with Jsoup
        val doc = Jsoup.parse(html)
        
        // Remove unwanted elements
        doc.select("script, style, nav, header, footer, iframe, noscript, svg").remove()
        
        // Remove common navigation and UI elements by class/id patterns
        doc.select("[class*='nav'], [class*='menu'], [class*='header'], [class*='footer'], [id*='nav'], [id*='menu']").remove()
        
        // Get main content area if exists
        val mainContent = doc.select("main, article, [role='main'], .content, #content").firstOrNull()
        val textSource = mainContent ?: doc.body()
        
        // Extract text with reasonable formatting
        return textSource.text()
            .replace("\\s+".toRegex(), " ") // Normalize whitespace
            .trim()
    }
}
```

### 3. Integration with Chat Service

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
    
    fun chatWithSources(userMessage: String, conversationId: String): Map<String, Any> {
        val response = chatClient.prompt()
            .user(userMessage)
            .advisors { advisor -> 
                advisor.param(ChatMemory.CONVERSATION_ID, conversationId)
            }
            .call()
            .chatResponse()
        
        // Extract sources from RAG context
        val documentContext = response.metadata[QuestionAnswerAdvisor.RETRIEVED_DOCUMENTS] as? List<Document>
        
        val sources = documentContext?.map { doc ->
            mapOf(
                "title" to doc.metadata["title"],
                "url" to doc.metadata["source"]
            )
        }?.distinctBy { it["url"] } ?: emptyList()
        
        return mapOf(
            "response" to response.result.output.content,
            "sources" to sources
        )
    }
}
```

### 4. Enhanced System Message

Update your system message to leverage the RAG context:

```kotlin
@Bean
fun chatClient(
    chatModel: ChatModel,
    vectorStore: VectorStore,
    chatMemory: ChatMemory
): ChatClient {
    return ChatClient.builder(chatModel)
        .defaultSystem("""
            You are a helpful airline loyalty program assistant specializing in Delta SkyMiles 
            and United MileagePlus programs. 
            
            Use the provided context from official airline websites to answer questions accurately.
            If the information is in the context, provide specific details about qualification 
            requirements, benefits, and program rules.
            
            If the answer is not in the provided context, acknowledge this and provide general 
            guidance or suggest checking the official airline website.
            
            Always be helpful, accurate, and cite information from the context when available.
        """.trimIndent())
        .defaultAdvisors(
            MessageChatMemoryAdvisor.builder(chatMemory).build(),
            QuestionAnswerAdvisor.builder(vectorStore)
                .searchRequest(
                    SearchRequest.builder()
                        .topK(5)
                        .similarityThreshold(0.7)
                        .build()
                )
                .build()
        )
        .build()
}
```

### 5. Controller Enhancement for Source Display

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
            val result = aiService.chatWithSources(query, conversationId)
            
            model.addAttribute("response", result["response"])
            model.addAttribute("sources", result["sources"])
        } catch (e: Exception) {
            model.addAttribute("error", e.message)
        }
        
        model.addAttribute("query", query)
        model.addAttribute("hasConversation", true)
        return "chat/index"
    }
}
```

### 6. Template Enhancement (Thymeleaf)

Update your `index.html` to display sources:

```html
<!-- Response section -->
<div th:if="${response}" class="response-section">
    <h3>Response:</h3>
    <p th:text="${response}"></p>
    
    <!-- Sources section -->
    <div th:if="${sources != null and !sources.isEmpty()}" class="sources-section">
        <h4>Sources:</h4>
        <ul>
            <li th:each="source : ${sources}">
                <a th:href="${source.url}" th:text="${source.title}" target="_blank"></a>
            </li>
        </ul>
    </div>
</div>
```

## Common Pitfalls to Avoid

1. **HTML Parsing**: Always clean HTML content to extract meaningful text. Remove scripts, styles, and navigation elements.

2. **Rate Limiting**: Be considerate when fetching web pages. Cache results and don't repeatedly fetch during development.

3. **Error Handling**: Web content loading can fail due to network issues, changed URLs, or blocked requests. Always handle exceptions gracefully.

4. **Content Extraction**: Different websites have different structures. The main content might be in `<main>`, `<article>`, or `<div class="content">`. Test your selectors.

5. **Vector Store Configuration**: `SimpleVectorStore` requires an `EmbeddingModel` in its constructor.

6. **Text Splitting**: Web pages can be large. Use appropriate chunk sizes (800-1000 tokens recommended).

7. **Metadata Access**: Use `document.metadata["key"]` in Kotlin or `document.getMetadata().get("key")` for metadata access.

8. **Memory Management**: `SimpleVectorStore` stores everything in memory. For production, consider persistent vector stores.

## Configuration (application.yml)

```yaml
spring:
  ai:
    openai:
      api-key: ${SPRING_AI_OPENAI_API_KEY}
      embedding:
        options:
          model: text-embedding-3-small
      chat:
        options:
          model: gpt-4
          temperature: 0.7

# Optional: HTTP client configuration
  http:
    client:
      connect-timeout: 10s
      read-timeout: 30s
```

## Testing Your Implementation

1. Start the application with `./gradlew bootRun`
2. Monitor logs for document loading messages:
   - "Starting to load airline documents from web..."
   - "Fetching content from: [URL]"
   - "Successfully loaded and embedded X document chunks"
3. Navigate to http://localhost:8080
4. Ask specific questions like:
   - "How do I qualify for Delta Gold Medallion status?"
   - "What are the requirements for United Premier Silver?"
   - "How many miles do I need to fly for Delta Platinum?"
5. Verify that:
   - Responses include accurate information from the loaded pages
   - Source URLs are displayed below responses
   - The assistant cites the correct airline program
6. Check error handling by stopping your internet connection temporarily

## Example Test Questions

- "What are the requirements to qualify for Delta Silver Medallion?"
- "How many PQP do I need for United Premier Gold?"
- "What's the difference between MQMs and MQDs for Delta?"
- "How do I earn Premier Qualifying Flights with United?"
- "Can I use credit card spending to qualify for status?"

## Performance Considerations

- Load documents once during application startup using `@PostConstruct`
- Use parallel streams for loading multiple URLs concurrently
- Configure appropriate timeouts for HTTP requests (10s connect, 30s read)
- Set reasonable chunk sizes (800 tokens) to balance context quality and retrieval speed
- Use `topK` of 5 and `similarityThreshold` of 0.7 for balanced results
- Consider implementing retry logic for failed URL fetches
- For production, cache loaded documents and refresh periodically

This enhancement will ground your airline loyalty assistant in authoritative, up-to-date information from official airline qualification pages while maintaining the lightweight, demonstration-focused nature of the application.
