---
title: "RAG with Open Source Vector Databases"
description: "Implementing Retrieval-Augmented Generation using Quarkus, LangChain4j, and open source vector databases."
complexity: "Intermediate"
use_case: "Knowledge Management"
technologies: ["Quarkus", "LangChain4j", "pgvector", "Redis"]
layout: architecture
---

# RAG with Open Source Vector Databases

Implement Retrieval-Augmented Generation (RAG) using Quarkus, LangChain4j, and open source vector databases to enhance AI responses with your business data.

## Architecture Overview

This pattern combines:
- **Quarkus** for the application runtime
- **LangChain4j** for AI orchestration
- **pgvector** or **Redis** for vector storage
- **OpenAI/Ollama** for language model inference

## Key Components

### 1. Document Ingestion Pipeline
- PDF/text document processing
- Chunking and embedding generation
- Vector storage with metadata

### 2. Query Processing
- User query embedding
- Similarity search in vector database
- Context assembly for LLM

### 3. Response Generation
- LLM prompt with retrieved context
- Response streaming
- Source attribution

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