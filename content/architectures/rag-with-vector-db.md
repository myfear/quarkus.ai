---
title: "Document Q&A with Retrieval-Augmented Generation (RAG)"
description: "A document question-answering system where an AI can answer user queries based on a private document corpus."
complexity: "Intermediate"
use_case: "Knowledge Management"
technologies: ["Quarkus", "LangChain4j", "pgvector", "Redis"]
layout: architecture
---

This follows the Retrieval-Augmented Generation (RAG) pattern. The application uses a LLM in conjunction with a vector database to provide relevant context from documents before generating answers. This design ensures accurate, context-aware responses by grounding the LLM in enterprise data instead of relying purely on the model’s own knowledge.

## Architecture Overview

This pattern combines:
- **Quarkus** for the application runtime
- **LangChain4j** for AI orchestration
- **pgvector** or **Redis** for vector storage
- **OpenAI/Ollama** for language model inference

## Key Components

* Quarkus service: Provides REST endpoints (or a UI) for users to ask questions. It orchestrates the RAG workflow.
* LangChain4j integration: Handles the workflow of embedding queries, searching the vector store, and invoking the LLM. It abstracts the low-level calls to the model.
* Vector database (pgvector/Postgres): Stores document embeddings for similarity search. When a question comes in, the service computes its embedding and finds relevant document chunks via pgvector.
* Local LLM via Ollama: The language model (e.g. Llama 2 or other) runs locally in an Ollama container. The Quarkus app sends the user’s question plus retrieved context to the LLM to generate an answer.
* No external API is needed, and data stays on-premise.
Supporting components: Quarkus Dev Services can spin up Postgres with pgvector. Optional health checks, monitoring (Micrometer/Prometheus) to observe AI service performance.

<img src="/assets/images/architectures/rag-with-vector/overview.png" alt="Architecture Overview" style="max-width: 100%; height: auto; display: block; margin: 0 auto;" />

## Implementation Guide

### Dependencies

```xml
<dependency>
    <groupId>io.quarkiverse.langchain4j</groupId>
    <artifactId>quarkus-langchain4j-openai</artifactId>
</dependency>
<dependency>
    <groupId>io.quarkiverse.langchain4j</groupId>
    <artifactId>quarkus-langchain4j-pgvector</artifactId>
</dependency>
```

### Configuration

```properties
# OpenAI Configuration
quarkus.langchain4j.openai.api-key=${OPENAI_API_KEY}

# pgvector Configuration
quarkus.datasource.db-kind=postgresql
quarkus.datasource.username=postgres
quarkus.datasource.password=postgres
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/vectordb
```

### Core Service Implementation

```java
@ApplicationScoped
public class RAGService {
    
    @Inject
    EmbeddingStore<TextSegment> embeddingStore;
    
    @Inject
    EmbeddingModel embeddingModel;
    
    @Inject
    ChatLanguageModel chatModel;
    
    public String query(String question) {
        // Generate embedding for the question
        Embedding questionEmbedding = embeddingModel.embed(question).content();
        
        // Find relevant documents
        List<EmbeddingMatch<TextSegment>> relevant = embeddingStore.findRelevant(
            questionEmbedding, 5, 0.7);
        
        // Construct context from relevant documents
        String context = relevant.stream()
            .map(match -> match.embedded().text())
            .collect(Collectors.joining("\n"));
        
        // Generate response with context
        String prompt = String.format(
            "Context: %s\n\nQuestion: %s\n\nAnswer:", 
            context, question);
        
        return chatModel.generate(prompt);
    }
}
```

## Deployment Considerations

### Scaling
- Horizontal scaling of query processing
- Vector database clustering
- Caching strategies for frequent queries

### Security
- API key management
- Data privacy and access control
- Audit logging for compliance

### Performance
- Embedding caching
- Connection pooling
- Batch processing for ingestion

## Best Practices

1. **Chunking Strategy**: Optimize chunk size for your domain
2. **Embedding Quality**: Choose appropriate embedding models
3. **Context Window**: Manage token limits effectively
4. **Monitoring**: Track query performance and accuracy

---

**Next Steps**: [Implement multi-tier context memory →](/architectures/multi-tier-context-memory/)