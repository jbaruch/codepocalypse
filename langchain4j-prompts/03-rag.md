# Airline Loyalty Assistant RAG Enhancement Task - In-Memory RAG

Enhance the Airline Loyalty Assistant web application by implementing a robust Retrieval Augmented Generation (RAG) system using LangChain4j, focusing on:

1. Add the ability to retrieve and process information from Delta and United loyalty program websites
2. Use LangChain4j's in-memory document storage for efficient information retrieval
3. Load documents from URLs asynchronously in parallel for improved performance
4. Enhance the quality and accuracy of responses with grounded information

## Technical Requirements

- Continue following **ALL** specifications in the `.windsurfrules` file
- Use ONLY LangChain4j's built-in document handling and RAG capabilities
- NEVER implement custom retrieval algorithms or similarity calculations
- Use the standard UrlDocumentLoader from LangChain4j for all document loading
- Maintain the existing simplified architecture and UI approach

## Implementation Guidelines

1. **Document Loading**:
   - Use LangChain4j's `UrlDocumentLoader` to load documents from airline websites
   - Create documents with proper source metadata for attribution
   - DO NOT implement custom document loading or processing logic

2. **Knowledge Base Construction**:
   - Use LangChain4j's `DocumentSplitters` for text chunking
   - Use LangChain4j's embedding models for vector encoding
   - Use `InMemoryEmbeddingStore` for vector storage
   - NEVER create custom embedding storage or similarity search algorithms

3. **RAG Implementation**:
   - Use ONLY standard LangChain4j methods for document retrieval
   - Use the embedding store's built-in retrieval methods for semantic search
   - DO NOT implement custom similarity calculations or ranking algorithms
   - Use standard LangChain4j approaches for combining retrieved information with LLM prompts

4. **Source Attribution**:
   - Include source URLs in document metadata
   - Track and display source information for retrieved documents
   - Format source attributions clearly in the UI

## Critical Implementation Details for LangChain4j 1.0.0-beta3

### 1. Required Components

- **Document Loading**: `UrlDocumentLoader` class with a `TextDocumentParser`
- **Metadata Handling**: `Metadata` class with `from()` factory method
- **Document Splitting**: `DocumentSplitters.recursive()` method with token count and overlap parameters
- **Embedding Model**: `OpenAiEmbeddingModel` configured with API key and timeout
- **Embedding Store**: `InMemoryEmbeddingStore<TextSegment>` for storing document vectors
- **Content Retriever**: `EmbeddingStoreContentRetriever` to connect AI services with the embedding store
- **Search Requests**: `EmbeddingSearchRequest` builder for configuring semantic search

### 2. Key API Methods to Use

- **For Document Loading**: `UrlDocumentLoader.load()` with proper URL encoding
- **For Document Creation**: `Document.from()` with text content and metadata
- **For Chunking**: `splitter.split()` to segment documents
- **For Embedding**: `embeddingModel.embedAll()` and `embeddingModel.embed()` 
- **For Storing Embeddings**: `embeddingStore.addAll()` for batch processing
- **For Searching**: `embeddingStore.search()` with a properly configured search request
- **For Metadata Access**: `metadata().getString()` method (NOT array-style access)
- **For Embedding Results**: Use `.content()` method (NOT `.get()`)

### 3. Integration Approaches

- **Option 1 - Content Retriever**: Configure AI services with a content retriever for automatic RAG
- **Option 2 - Manual Integration**: Retrieve relevant documents and manually incorporate them into prompts

## Common Pitfalls to Avoid

1. **API Version Mismatches**: LangChain4j 1.0.0-beta3 has specific method signatures and package structures. Always verify imports and method calls.

2. **Metadata Handling**: Use `metadata().getString("key")` instead of array-style access `metadata()["key"]`.

3. **Embedding Results**: Use `content()` method on embedding results, not `get()`.

4. **URL Encoding**: Always encode URLs properly using `URLEncoder.encode()` before passing to UrlDocumentLoader.

5. **Batch Processing**: Use `addAll()` for adding multiple embeddings to the store instead of individual `add()` calls.

6. **Error Handling**: Always implement proper error handling for document loading and embedding operations as they may fail.

7. **Async Loading**: Use `CompletableFuture.runAsync` for non-blocking document loading.

## Testing Your Implementation

1. Start the application with `./gradlew quarkusDev`
2. Navigate to http://localhost:8080
3. Ask questions about Delta SkyMiles or United MileagePlus programs
4. Verify that responses include information from the loaded documents
5. Check logs to confirm document loading, chunking, and retrieval are working correctly
