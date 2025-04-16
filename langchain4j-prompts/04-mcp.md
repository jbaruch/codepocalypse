# Airline Loyalty Assistant MCP Enhancement Task

Enhance the Airline Loyalty Assistant web application by implementing Model Context Protocol (MCP) support using LangChain4j, focusing on:

1. Using LangChain4j to run the Google Flights MCP server directly
2. Enabling the assistant to answer real-time flight availability questions
3. Maintaining the simplified architecture while adding powerful contextual capabilities
4. Using LangChain4j's MCP client capabilities for seamless integration

## Technical Requirements

- Continue following **ALL** specifications in the user rules file
- Use ONLY LangChain4j's built-in MCP capabilities
- Maintain the existing simplified architecture and UI approach
- Use LangChain4j to run the Google Flights MCP server directly
- Use Kotlin for all implementation code

## Implementation Guidelines

1. **MCP Server Configuration**:
   - We'll use existing Google Flights MCP server: https://github.com/salamentic/google-flights-mcp. The code is checked out and available on the local machine, and can be run using the following configuration:
   ```json
       "flight-planner": {
      "command": "/opt/homebrew/bin/python3",
      "args": [
        "/Users/jbaruch/Projects/google-flights-mcp/src/flights-mcp-server.py"
      ],
      "env": {
        "PYTHONPATH": "/Users/jbaruch/Projects/google-flights-mcp:/opt/homebrew/lib/python3.13/site-packages:/opt/homebrew/opt/python@3.13/Frameworks/Python.framework/Versions/3.13/lib/python3.13/site-packages",
        "PATH": "/usr/local/bin:/usr/bin:/bin"
      }
    }
   ```
   - Use LangChain4j's `StdioMcpTransport` to run the Google Flights MCP server directly
   - Configure the MCP server with the correct executable and arguments from the json config above:
     ```kotlin
     val transport = StdioMcpTransport.Builder()
         .command(listOf("python3", "/Users/jbaruch/Projects/google-flights-mcp/src/flights-mcp-server.py"))
         .logEvents(true)
         .build()
     ```
   - Set up proper error handling for server startup issues
   - DO NOT implement custom MCP server logic

2. **Integration with Existing Assistant**:
   - Extend the current assistant to use MCP for flight-related queries
   - Maintain the existing UI and form submission approach
   - Ensure seamless transition between general queries and flight-specific queries
   - Use a detection mechanism to route flight queries to the MCP-enabled model

3. **Query Processing**:
   - Detect flight-related queries (e.g., containing flight numbers, routes, dates)
   - Format queries appropriately for the MCP server
   - Parse and format responses for user-friendly display
   - Handle errors gracefully with appropriate user feedback

4. **Response Handling**:
   - Process structured data from the MCP server
   - Format flight information in a clear, readable format
   - Include all relevant details (times, prices, airlines, etc.)
   - Maintain attribution to the data source

## Common Pitfalls to Avoid

1. **API Version Mismatches**: LangChain4j 1.0.0-beta3 has specific method signatures and package structures. Always verify imports and method calls.

2. **Server Startup Handling**: Implement proper error handling for MCP server startup issues.

3. **Query Detection**: Ensure accurate detection of flight-related queries to route to the MCP-enabled model.

4. **Response Formatting**: Properly parse and format structured data from the MCP server.

5. **Error Handling**: Implement graceful fallbacks if the MCP server fails to start or respond.

6. **Timeout Configuration**: Set appropriate timeouts for MCP operations.

## Sample Implementation Code

```kotlin
// Sample MCP configuration
val transport = StdioMcpTransport.Builder()
    .command(listOf("python", "flight_planner_server.py"))
    .logEvents(true)
    .build()

val mcpClient = DefaultMcpClient.Builder()
    .transport(transport)
    .build()

val toolProvider = McpToolProvider.builder()
    .mcpClients(listOf(mcpClient))
    .build()

// Configure AI service with MCP tool provider
val aiService = AiServices.builder(AssistantService::class.java)
    .chatLanguageModel(chatModel)
    .toolProvider(toolProvider)
    .build()

// Sample query processing
fun processQuery(query: String): String {
    // Process using the AI service with MCP tools
    try {
        return aiService.chat(query)
    } catch (e: Exception) {
        log.error("Error processing query", e)
        return "I'm sorry, I couldn't process your query at this time."
    }
}

// Interface for the AI service
interface AssistantService {
    @SystemMessage("""
        You are an airline loyalty assistant that can help with flight information.
        When asked about flights, use the available MCP tools to provide accurate information.
    """)
    fun chat(userMessage: String): String
}
```

## Testing Your Implementation

1. Start the application with `./gradlew quarkusDev`
2. Navigate to http://localhost:8080
3. Ask questions like:
   - "What flights are available from NYC to SFO on 2025-04-15?"
   - "Find me the cheapest flight from Boston to Chicago next week"
   - "Are there any direct flights from LAX to JFK tomorrow?"
4. Verify that responses include accurate flight information
5. Check logs to confirm proper MCP server operation

## Expected Behavior

When a user asks about flights, the system should:
1. Detect the flight-related query
2. Use the LangChain4j MCP integration to process the query
3. Retrieve real-time flight information
4. Format and display the results clearly
5. Handle any errors gracefully

For non-flight queries, the system should continue to use the existing chat functionality without disruption.

## Available MCP Tools and Resources

The Google Flights MCP server provides the following tools:

- `search_one_way_flights`: Search for one-way flights between airports
- `search_round_trip_flights`: Search for round-trip flights between airports  
- `create_travel_plan`: Generate a comprehensive travel plan

And the following resources:

- `airport_codes://{query}`: Get airport code information based on a search query

Make sure to implement support for these capabilities in your integration.
