---
title: "Memory-Augmented Conversational AI Agent"
description: "This pattern enables an AI chatbot to remember and utilize past interactions across multiple conversation turns or sessions."
complexity: "Expert"
use_case: "Knowledge Management"
technologies: ["Quarkus", "LangChain4j", "pgvector", "Pgvector", "PostgresQL"]
layout: architecture
---

The use case is a context-aware conversational agent (e.g., a customer support bot or personal assistant) that can carry on long, coherent dialogues by retaining important information from earlier exchanges. Instead of treating each query in isolation, the agent builds long-term semantic memory of the conversation so it can refer back to prior user inputs, events, or facts. Built on Quarkus and LangChain4j, this architecture transforms raw conversations into structured, searchable memory.

As a result, the agent can provide intelligent, contextually relevant responses even as dialogues grow, avoiding repetition and improving user experience. In essence, the pattern combines Quarkus’s cloud-native Java stack with LangChain4j’s AI abstractions to create an AI service that “remembers” – storing and retrieving conversation history using embeddings and a vector database (pgvector) for semantic recall.

<img src="/assets/images/architectures/multi-tier-context-memory/overview.png" alt="Architecture Overview" style="max-width: 100%; height: auto; display: block; margin: 0 auto;" />

## Architecture Overview

This pattern combines:
- **Quarkus** for the application runtime
- **LangChain4j** for AI orchestration
- **PostgresQL/Pgvector** for language model inference

Got it. I’ll review the `ai-memory` repository and extract a high-level architecture pattern, consistent with the abstraction level of the other reference architectures. I’ll include a clear description, key components, and a proposed diagram layout suitable for featuring on quarkus.ai.

I’ll follow up with the results shortly.


## Key Components

**Quarkus Application Layers:** The solution is implemented as a standard Quarkus application with a RESTful or WebSocket interface for client interaction. For example, a JAX-RS resource or WebSocket endpoint accepts user messages and returns AI-generated replies. This layer delegates to an AI Service (a CDI bean) that encapsulates the AI conversation logic. The Quarkus runtime handles dependency injection, configuration, and efficient request handling, ensuring the AI agent can scale and integrate with other Java components (database, scheduler, etc.) as needed.

**LangChain4j AI Service:** At the core is a LangChain4j-powered AI Service, defined as a Java interface with the `@RegisterAiService` annotation. Quarkus LangChain4j generates an implementation of this interface that interacts with a Large Language Model (LLM). The interface might look like:

```java
@RegisterAiService
@ApplicationScoped
public interface ChatAgentService {
    String converse(@MemoryId String sessionId, String userMessage);
}
```

Here, `converse(...)` represents a chat interaction method. The `@MemoryId` on the session parameter tells Quarkus to maintain a distinct memory context per user or session. This prevents cross-talk between conversations and scopes the memory to each user. The AI service method can also use annotations like `@UserMessage` and `@SystemMessage` to frame prompts. For instance, a system prompt could set the assistant’s persona or instructions (e.g., *“You are a helpful travel assistant.”*), and the user message is the dynamic part of the prompt template.

**AI Model Integration:** The LangChain4j Quarkus extension simplifies connecting to LLMs. In this pattern, an LLM (such as a GPT-4 class model or a local Llama2 via Ollama) is configured in `application.properties` and accessed through the AI service. Quarkus.ai supports various model backends; in our repository example, it uses **Ollama** (for running local models) via the `quarkus-langchain4j-ollama` extension. The model processes each prompt and generates a completion. Because LLM calls are stateless, the service must provide **context (memory)** with each request. The integration handles prompt construction and API calls to the model, returning the assistant’s response as a string.

**Conversation Memory (Short-Term):** LangChain4j automatically maintains a **chat memory** for the AI service to give the LLM needed context on each turn. Every time the user calls `ChatAgentService.converse(...)`, the extension will inject recent conversation history into the prompt (up to a configurable window). By default this is an in-memory chat log of the last N messages (10 by default, adjustable via config). This short-term memory allows the AI to understand follow-up questions or references (resolving pronouns like “it” or “that”) based on earlier messages. The memory is tied to the CDI scope of the AI service: for example, if the service is `@ApplicationScoped`, you must use `@MemoryId` to isolate each user’s chat; in a web app with `@SessionScoped` services, each user session gets its own memory by design.

**Long-Term Semantic Memory (Vector DB):** To augment the agent beyond the short chat window, the pattern introduces a **persistent semantic memory store** using Postgres with the **pgvector** extension. As conversations progress, older messages or extracted facts are **embedded** into high-dimensional vectors and stored in the database for long-term retention. The Quarkus LangChain4j PGVector integration (`quarkus-langchain4j-pgvector`) provides an embedding store and retrieval capability backed by Hibernate + Panache for ease of use. Essentially, each significant piece of conversation (e.g. a user query, an assistant answer, or a summary of a dialogue chunk) is saved along with its vector embedding. This makes the memory **searchable by semantic similarity**. For instance, the agent could later retrieve relevant past content even if the user doesn’t repeat the exact keywords (the vector store can find semantically related information).

Under the hood, an **Embeddings model** (possibly the LLM itself or a dedicated embedding model) converts text to numeric vectors. A **Vector Store** (pgvector in Postgres) indexes these vectors. On each new user query, the system can embed the query and query the vector store for similar content from the memory, effectively performing a **semantic recall** of long-term memory. This retrieved context is then fed into the LLM prompt (for example, injected as additional system message or context text) alongside the recent chat history. The pattern thus merges Retrieval-Augmented Generation (RAG) techniques with conversational memory – the agent remembers by **retrieving** stored knowledge relevant to the current query, rather than naively replaying the entire history.

**Memory Scope and Lifecycle:** The memory system operates on two levels:

* *Session scope (isolated memory):* The use of `@MemoryId` or session-scoped services ensures each conversation has its own memory thread. For example, memory entries in the vector DB include a session or user identifier, so the agent doesn’t mix up different users' data. The application is responsible for providing a consistent ID (like user ID or chat session ID) whenever calling the AI service.
* *Memory retention policy:* Short-term memory is typically bounded (e.g., last 10 messages) to fit within the model’s token limit. Older messages can be summarized or removed from the immediate context (LangChain4j can use a *MessageWindow* or *TokenWindow* strategy to evict old messages). When evicted, those messages don’t disappear entirely – they can be persisted in the long-term store. Optionally, a background **summarization** job could periodically condense old conversations and save summaries (keeping memory “self-organizing” and manageable). This way, the vector DB holds either raw past messages or their summaries, and the agent can recall high-level context later without exceeding token limits.

**Additional Quarkus Components:** The architecture may include a Panache repository or an entity for memory records (e.g., a `MemoryEntry` entity with fields for sessionId, content text, embedding vector, timestamp, etc.). Quarkus Data (Hibernate ORM with Panache) simplifies CRUD operations to save new memory entries and query existing ones. A scheduler (via `@Scheduled`) might run tasks like cleaning up old data or triggering summarization. All components are lightweight and benefit from Quarkus dev services (which can spin up the Postgres with pgvector automatically in dev mode) and Quarkus Dev UI for monitoring database state.

## Conversational Flow with Memory 

Below is a step-by-step flow of how user input and memory interplay across the system's components. This also describes a logical diagram of the architecture:

1. **User Interaction:** A user sends a message (e.g. via an HTTP POST or WebSocket message) to the Quarkus application. For example, the user asks: *“I’m traveling to Paris. What should I pack?”*.

2. **API Layer (Quarkus Endpoint):** The incoming message is received by a JAX-RS resource or controller. The endpoint identifies the user/session (e.g., from a token or session cookie) and invokes the `ChatAgentService.converse(sessionId, userMessage)` method. This call is handled by the LangChain4j-generated implementation behind the interface.

3. **Memory Retrieval:** Upon invocation, LangChain4j **rehydrates the context** for this session. It loads the recent chat history (previous user and assistant messages in this session) from memory. In addition, a semantic lookup is performed against the **vector store**: the new user query is embedded into a vector, and a similarity search finds any stored memory entries relevant to “traveling to Paris” or related packing advice. For example, if in an earlier session the user discussed a trip to Paris or preferences about travel gear, those vectorized memories would be retrieved as pertinent context.

4. **Prompt Assembly:** The AI service composes a prompt for the LLM that includes:

   * A **system message** defining the assistant’s role and any necessary instructions (e.g., *“You are a travel assistant AI that recalls user preferences.”*).
   * **Relevant context** injected from memory: this could be prior conversation snippets or facts fetched in step 3. These might be included as additional system or user messages (for instance: *“(Recall): Earlier the user mentioned they dislike heavy luggage.”*).
   * The **current user message** asking about packing for Paris (as the latest user message in the prompt sequence).

5. **LLM Processing:** The LangChain4j service sends the assembled prompt to the configured LLM model (via the Ollama backend or an API call, depending on configuration). The model generates an answer, taking into account both the immediate question and the provided context from memory. For example, the LLM might answer: *“Since you mentioned you prefer to travel light and Paris can be chilly in spring, bring a warm jacket and comfortable shoes...”*.

6. **Response Handling:** The AI service returns the model’s answer to the Quarkus endpoint, which in turn sends it back to the user (HTTP response or WebSocket message). The user sees the answer in the chat UI or client.

7. **Memory Update:** As the final step of the call, the system updates its memory:

   * The **short-term chat log** is appended with the new user message and the assistant’s answer, so that they will be part of the context for the next turn. Quarkus LangChain4j manages this chat history automatically in memory (or in a distributed scenario, via a shared `ChatMemoryStore`).
   * The **long-term store** is also updated. The new user query and/or the assistant’s response can be vectorized and saved into PostgreSQL (pgvector) via the Panache repository. This ensures that key information from this exchange becomes part of the persistent semantic memory. For instance, the fact that the user is traveling to Paris and likes to travel light may be saved as a vector embedding in the memory store. Over time, the vector database accumulates a knowledge base of user-specific facts and past dialog summaries.

8. **Iteration:** The user can ask follow-up questions (e.g., *“How about in the summer?”*). The cycle repeats: the agent will retrieve the latest context (it now knows the conversation about Paris, packing preferences, etc.) and respond accordingly, maintaining a coherent thread. If the conversation continues extensively, older messages might be dropped from the immediate prompt (to avoid hitting token limits), but thanks to the long-term memory store, the agent can always fetch relevant details later by semantic search rather than forgetting them completely.

Overall, the Memory-Augmented AI Agent architecture provides a blueprint for building intelligent Java-based chatbots that **learn from each conversation**. By combining immediate conversational memory with a long-term semantic vector store, the agent can handle context-aware interactions that feel natural to users – all within a Quarkus application that architects can easily integrate into enterprise systems. This reusable pattern can be applied to any AI assistant requiring long-term knowledge retention, using Quarkus and LangChain4j as the foundation for a robust, memory-enabled AI service.


