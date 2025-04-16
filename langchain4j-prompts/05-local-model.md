# Switching to a Local Mistral Model

This guide shows how to switch from OpenAI's ChatGPT to a local Mistral model running in Docker using LangChain4j.

## Configuration

Replace your existing OpenAI model configuration with the following Mistral configuration:

```kotlin
ChatLanguageModel model = OpenAiChatModel.builder()
  .apiKey("not needed")
  .baseUrl("http://localhost:12434/engines/llama.cpp/v1")
  .modelName("ai/mistral")
  .build();
```

## Implementation Steps

1. Make sure you have Docker installed and running on your machine.

2. Start the local Mistral model container (if not already running).

3. Update your model configuration in your application code as shown above.

4. The rest of your LangChain4j code remains the same - the library abstracts away the differences between model providers.

## Notes

- No API key is required for the local model, but we still need to provide a placeholder value.
- The base URL points to the local Docker container running on port 12434.
- The model name is set to "ai/mistral".
- This configuration uses the OpenAI-compatible API that the local Mistral container exposes.

## Example Usage

Your existing code for creating chat messages and getting responses will work the same way:

```kotlin
Messages messages = Messages.from(
    SystemMessage.from("You are a helpful assistant."),
    UserMessage.from("Tell me about Mistral AI models.")
);

String response = model.generate(messages).content();
```

The change is transparent to the rest of your application, which is one of the benefits of using LangChain4j's abstraction layer.
