# Part 075: Spring AI — Integrating AI into Spring Boot

Spring AI provides a unified, portable API for integrating AI models into Spring Boot applications. This part builds an AI-powered customer support chatbot using OpenAI/Claude with RAG, streaming, and function calling.

---

## Table of Contents

1. [Spring AI Overview](#overview)
2. [Project Setup](#setup)
3. [ChatClient Basics](#chatclient)
4. [Prompt Engineering in Java](#prompts)
5. [Streaming Responses](#streaming)
6. [Embeddings and Vector Similarity](#embeddings)
7. [Vector Store Integration](#vector-store)
8. [RAG Pattern](#rag)
9. [Function Calling / Tool Use](#tools)
10. [Image Generation](#image-gen)
11. [Cost Management](#cost)
12. [AI Content Validation](#validation)
13. [Real Example: Customer Support Chatbot](#chatbot)

---

## 1. Spring AI Overview {#overview}

Spring AI abstracts over multiple AI providers (OpenAI, Anthropic, Azure OpenAI, Ollama, etc.) with a consistent API.

**Key Components:**
- `ChatClient` — send messages, get responses
- `EmbeddingModel` — convert text to vectors
- `VectorStore` — store and search vector embeddings
- `ChatMemory` — maintain conversation history
- `Advisor` — intercept and augment chat interactions

---

## 2. Project Setup {#setup}

```xml
<!-- pom.xml -->
<project>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>ai-support-chatbot</artifactId>
    <version>1.0.0</version>

    <properties>
        <spring-ai.version>1.0.0-M3</spring-ai.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-webflux</artifactId>
        </dependency>

        <!-- Spring AI with OpenAI -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
            <version>${spring-ai.version}</version>
        </dependency>

        <!-- Spring AI with Anthropic Claude (alternative) -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-anthropic-spring-boot-starter</artifactId>
            <version>${spring-ai.version}</version>
        </dependency>

        <!-- PGVector store -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
            <version>${spring-ai.version}</version>
        </dependency>

        <!-- Redis vector store (alternative) -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-redis-store-spring-boot-starter</artifactId>
            <version>${spring-ai.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.ai</groupId>
                <artifactId>spring-ai-bom</artifactId>
                <version>${spring-ai.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <repositories>
        <repository>
            <id>spring-milestones</id>
            <url>https://repo.spring.io/milestone</url>
        </repository>
    </repositories>
</project>
```

```yaml
# src/main/resources/application.yml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
          max-tokens: 2048
      embedding:
        options:
          model: text-embedding-3-small

    # OR Anthropic Claude
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-3-5-sonnet-20241022
          max-tokens: 2048

    vectorstore:
      pgvector:
        index-type: HNSW
        distance-type: COSINE_DISTANCE
        dimensions: 1536

  datasource:
    url: jdbc:postgresql://localhost:5432/aidb
    username: ${DB_USER}
    password: ${DB_PASSWORD}
```

---

## 3. ChatClient Basics {#chatclient}

```java
// src/main/java/com/example/ai/ChatService.java
package com.example.ai;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.model.ChatResponse;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.prompt.SystemPromptTemplate;
import org.springframework.ai.chat.messages.*;
import org.springframework.ai.openai.OpenAiChatOptions;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

@Service
public class ChatService {

    private final ChatClient chatClient;

    public ChatService(ChatClient.Builder chatClientBuilder) {
        // Configure a default system prompt
        this.chatClient = chatClientBuilder
            .defaultSystem("You are a helpful assistant for a software company.")
            .build();
    }

    /**
     * Simple one-shot question and answer
     */
    public String ask(String question) {
        return chatClient.prompt()
            .user(question)
            .call()
            .content();
    }

    /**
     * With custom system prompt per request
     */
    public String askWithContext(String systemContext, String question) {
        return chatClient.prompt()
            .system(systemContext)
            .user(question)
            .call()
            .content();
    }

    /**
     * Get full response including metadata
     */
    public ChatResponse askFull(String question) {
        return chatClient.prompt()
            .user(question)
            .call()
            .chatResponse();
    }

    /**
     * Use PromptTemplate for parameterized prompts
     */
    public String generateProductDescription(String productName, String features) {
        return chatClient.prompt()
            .user(u -> u.text("""
                Write a compelling product description for {productName}.
                Key features: {features}
                Keep it under 100 words, professional and engaging.
                """)
                .param("productName", productName)
                .param("features", features)
            )
            .call()
            .content();
    }

    /**
     * Override model settings per request
     */
    public String askWithHighCreativity(String prompt) {
        return chatClient.prompt()
            .user(prompt)
            .options(OpenAiChatOptions.builder()
                .withTemperature(1.2f)
                .withModel("gpt-4o")
                .withMaxTokens(1024)
                .build()
            )
            .call()
            .content();
    }

    /**
     * Multi-turn conversation using message history
     */
    public String chat(List<Message> history, String newMessage) {
        return chatClient.prompt()
            .messages(history)
            .user(newMessage)
            .call()
            .content();
    }
}
```

---

## 4. Prompt Engineering in Java {#prompts}

```java
// src/main/java/com/example/ai/PromptTemplates.java
package com.example.ai;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.prompt.PromptTemplate;
import org.springframework.core.io.ClassPathResource;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
public class PromptTemplates {

    private final ChatClient chatClient;

    public PromptTemplates(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    /**
     * Structured output with JSON extraction
     */
    public ProductInfo extractProductInfo(String userMessage) {
        return chatClient.prompt()
            .system("""
                You are a data extraction assistant.
                Extract product information from user messages.
                Always respond with valid JSON matching the ProductInfo schema.
                """)
            .user(userMessage)
            .call()
            .entity(ProductInfo.class);  // Spring AI auto-parses JSON to POJO
    }

    /**
     * Few-shot prompting
     */
    public String classifyIntent(String userMessage) {
        String fewShotPrompt = """
            Classify the following customer message into one of these categories:
            BILLING, TECHNICAL, ACCOUNT, GENERAL
            
            Examples:
            "My invoice is wrong" → BILLING
            "App crashes on login" → TECHNICAL
            "Change my password" → ACCOUNT
            "What are your hours" → GENERAL
            
            Message: {message}
            Category:""";

        return chatClient.prompt()
            .user(u -> u.text(fewShotPrompt).param("message", userMessage))
            .call()
            .content()
            .trim();
    }

    /**
     * Chain-of-thought prompting
     */
    public String solveWithReasoning(String problem) {
        return chatClient.prompt()
            .user("""
                Solve this step by step:
                
                Problem: %s
                
                Instructions:
                1. Break down the problem
                2. Identify key constraints
                3. Work through each step
                4. Validate your answer
                5. State the final answer clearly
                """.formatted(problem))
            .call()
            .content();
    }

    /**
     * Load prompt template from file
     */
    public String processWithFileTemplate(Map<String, Object> variables) {
        PromptTemplate template = new PromptTemplate(
            new ClassPathResource("prompts/customer-support.st")
        );
        return chatClient.prompt(template.create(variables))
            .call()
            .content();
    }

    record ProductInfo(String name, String category, Double price, String description) {}
}
```

```
// src/main/resources/prompts/customer-support.st
You are a helpful customer support agent for {companyName}.
Customer name: {customerName}
Account tier: {accountTier}
Previous tickets: {previousTicketCount}

Current issue:
{issue}

Provide a helpful, empathetic response. If it's a technical issue, provide steps.
For billing issues, explain the process. Always offer to escalate if needed.
```

---

## 5. Streaming Responses {#streaming}

```java
// src/main/java/com/example/ai/StreamingChatService.java
package com.example.ai;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Service;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;

@Service
public class StreamingChatService {

    private final ChatClient chatClient;

    public StreamingChatService(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("You are a helpful assistant.")
            .build();
    }

    /**
     * Stream response as text/event-stream
     */
    public Flux<String> streamAnswer(String question) {
        return chatClient.prompt()
            .user(question)
            .stream()
            .content();
    }

    /**
     * Stream full ChatResponse objects (includes metadata per chunk)
     */
    public Flux<org.springframework.ai.chat.model.ChatResponse> streamFullResponse(String question) {
        return chatClient.prompt()
            .user(question)
            .stream()
            .chatResponse();
    }
}

@RestController
@RequestMapping("/api/chat")
class ChatStreamController {

    private final StreamingChatService streamingService;

    ChatStreamController(StreamingChatService streamingService) {
        this.streamingService = streamingService;
    }

    /**
     * SSE endpoint — browser connects and receives tokens as they stream
     */
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamChat(@RequestParam String question) {
        return streamingService.streamAnswer(question);
    }

    /**
     * Collect all tokens and return complete response
     */
    @PostMapping("/ask")
    public Flux<String> askAndStream(@RequestBody ChatRequest request) {
        return streamingService.streamAnswer(request.question());
    }

    record ChatRequest(String question, String sessionId) {}
}
```

---

## 6. Embeddings and Vector Similarity {#embeddings}

```java
// src/main/java/com/example/ai/EmbeddingService.java
package com.example.ai;

import org.springframework.ai.embedding.EmbeddingModel;
import org.springframework.ai.embedding.EmbeddingResponse;
import org.springframework.stereotype.Service;

import java.util.Arrays;
import java.util.List;

@Service
public class EmbeddingService {

    private final EmbeddingModel embeddingModel;

    public EmbeddingService(EmbeddingModel embeddingModel) {
        this.embeddingModel = embeddingModel;
    }

    /**
     * Embed a single text into a vector
     */
    public float[] embed(String text) {
        return embeddingModel.embed(text);
    }

    /**
     * Embed multiple texts in one API call (batch)
     */
    public List<float[]> embedBatch(List<String> texts) {
        EmbeddingResponse response = embeddingModel.embedForResponse(texts);
        return response.getResults().stream()
            .map(r -> r.getOutput())
            .toList();
    }

    /**
     * Compute cosine similarity between two vectors
     */
    public double cosineSimilarity(float[] vec1, float[] vec2) {
        if (vec1.length != vec2.length) {
            throw new IllegalArgumentException("Vectors must have same dimension");
        }

        double dotProduct = 0.0;
        double norm1 = 0.0;
        double norm2 = 0.0;

        for (int i = 0; i < vec1.length; i++) {
            dotProduct += vec1[i] * vec2[i];
            norm1 += vec1[i] * vec1[i];
            norm2 += vec2[i] * vec2[i];
        }

        if (norm1 == 0 || norm2 == 0) return 0.0;
        return dotProduct / (Math.sqrt(norm1) * Math.sqrt(norm2));
    }

    /**
     * Find most similar text from a list
     */
    public String findMostSimilar(String query, List<String> candidates) {
        float[] queryVector = embed(query);
        List<float[]> candidateVectors = embedBatch(candidates);

        double bestScore = -1;
        int bestIndex = 0;

        for (int i = 0; i < candidateVectors.size(); i++) {
            double score = cosineSimilarity(queryVector, candidateVectors.get(i));
            if (score > bestScore) {
                bestScore = score;
                bestIndex = i;
            }
        }

        return candidates.get(bestIndex);
    }
}
```

---

## 7. Vector Store Integration {#vector-store}

```java
// src/main/java/com/example/ai/KnowledgeBaseService.java
package com.example.ai;

import org.springframework.ai.document.Document;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.stereotype.Service;

import java.util.*;

@Service
public class KnowledgeBaseService {

    private final VectorStore vectorStore;

    public KnowledgeBaseService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    /**
     * Add documents to the vector store
     * Spring AI auto-embeds them using the configured EmbeddingModel
     */
    public void addDocuments(List<KnowledgeDocument> knowledgeDocs) {
        List<Document> documents = knowledgeDocs.stream()
            .map(doc -> {
                Map<String, Object> metadata = new HashMap<>();
                metadata.put("source", doc.source());
                metadata.put("category", doc.category());
                metadata.put("id", doc.id());

                return new Document(doc.content(), metadata);
            })
            .toList();

        vectorStore.add(documents);
    }

    /**
     * Semantic search — finds documents by meaning, not keywords
     */
    public List<Document> search(String query, int topK) {
        return vectorStore.similaritySearch(
            SearchRequest.query(query).withTopK(topK)
        );
    }

    /**
     * Filtered semantic search
     */
    public List<Document> searchByCategory(String query, String category, int topK) {
        return vectorStore.similaritySearch(
            SearchRequest.query(query)
                .withTopK(topK)
                .withFilterExpression("category == '" + category + "'")
                .withSimilarityThreshold(0.7) // Only return if similarity > 70%
        );
    }

    /**
     * Load FAQ documents into vector store
     */
    public void loadFaqDocuments(List<FaqEntry> faqs) {
        List<Document> documents = faqs.stream()
            .map(faq -> {
                // Combine question and answer for richer embedding
                String content = "Q: " + faq.question() + "\nA: " + faq.answer();
                return new Document(
                    content,
                    Map.of(
                        "question", faq.question(),
                        "answer", faq.answer(),
                        "category", faq.category(),
                        "type", "faq"
                    )
                );
            })
            .toList();

        vectorStore.add(documents);
    }

    record KnowledgeDocument(String id, String content, String source, String category) {}
    record FaqEntry(String question, String answer, String category) {}
}
```

### Vector Store Schema (PostgreSQL with pgvector)

```sql
-- src/main/resources/schema-vector.sql
-- Enable pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Spring AI creates this table automatically when using PgVectorStore
-- But you can also create it manually:
CREATE TABLE IF NOT EXISTS vector_store (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    content TEXT NOT NULL,
    metadata JSONB,
    embedding vector(1536)  -- 1536 dimensions for OpenAI text-embedding-3-small
);

-- HNSW index for fast approximate nearest neighbor search
CREATE INDEX IF NOT EXISTS vector_store_embedding_idx
    ON vector_store USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

---

## 8. RAG Pattern (Retrieval-Augmented Generation) {#rag}

```java
// src/main/java/com/example/ai/RagService.java
package com.example.ai;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.QuestionAnswerAdvisor;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.stereotype.Service;

@Service
public class RagService {

    private final ChatClient chatClient;

    public RagService(ChatClient.Builder builder, VectorStore vectorStore) {
        this.chatClient = builder
            .defaultSystem("""
                You are a helpful customer support agent.
                Answer questions based on the provided context.
                If the answer is not in the context, say you don't know
                and offer to escalate to a human agent.
                Always be polite and professional.
                """)
            // QuestionAnswerAdvisor automatically retrieves relevant docs and adds to prompt
            .defaultAdvisors(new QuestionAnswerAdvisor(
                vectorStore,
                SearchRequest.defaults()
                    .withTopK(5)
                    .withSimilarityThreshold(0.6)
            ))
            .build();
    }

    /**
     * RAG-powered question answering
     * The advisor automatically:
     * 1. Embeds the question
     * 2. Retrieves top-K similar documents
     * 3. Adds them as context to the prompt
     * 4. The model answers based on context
     */
    public String answer(String question) {
        return chatClient.prompt()
            .user(question)
            .call()
            .content();
    }
}
```

```java
// src/main/java/com/example/ai/CustomRagService.java
package com.example.ai;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.document.Document;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

/**
 * Manual RAG implementation for more control over retrieval and prompt construction
 */
@Service
public class CustomRagService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public CustomRagService(ChatClient.Builder builder, VectorStore vectorStore) {
        this.chatClient = builder.build();
        this.vectorStore = vectorStore;
    }

    public String answerWithRag(String question, String customerId) {
        // Step 1: Retrieve relevant documents
        List<Document> relevantDocs = vectorStore.similaritySearch(
            SearchRequest.query(question)
                .withTopK(5)
                .withSimilarityThreshold(0.65)
        );

        if (relevantDocs.isEmpty()) {
            return "I don't have specific information about that. "
                + "Let me connect you with a human agent.";
        }

        // Step 2: Build context from retrieved documents
        String context = relevantDocs.stream()
            .map(Document::getContent)
            .collect(Collectors.joining("\n\n---\n\n"));

        // Step 3: Construct augmented prompt
        String systemPrompt = """
            You are a customer support agent.
            Use the following knowledge base articles to answer the question.
            If the answer isn't in the provided context, say so clearly.
            
            CONTEXT:
            %s
            
            INSTRUCTIONS:
            - Answer based only on the provided context
            - Be concise and helpful
            - If multiple articles are relevant, synthesize them
            - Cite the source if metadata contains it
            """.formatted(context);

        // Step 4: Generate answer
        String answer = chatClient.prompt()
            .system(systemPrompt)
            .user(question)
            .call()
            .content();

        // Step 5: Log for quality monitoring
        logRagInteraction(customerId, question, relevantDocs.size(), answer);

        return answer;
    }

    private void logRagInteraction(String customerId, String question,
                                    int docsRetrieved, String answer) {
        System.out.printf("RAG: customer=%s, docs_retrieved=%d, question_length=%d%n",
            customerId, docsRetrieved, question.length());
    }
}
```

---

## 9. Function Calling / Tool Use {#tools}

```java
// src/main/java/com/example/ai/tools/CustomerTools.java
package com.example.ai.tools;

import com.example.ai.CustomerService;
import com.example.ai.OrderService;
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.stereotype.Component;

import java.util.List;

/**
 * Tools (functions) that the AI can call during a conversation.
 * Spring AI handles the function call/response loop automatically.
 */
@Component
public class CustomerTools {

    private final CustomerService customerService;
    private final OrderService orderService;

    public CustomerTools(CustomerService customerService, OrderService orderService) {
        this.customerService = customerService;
        this.orderService = orderService;
    }

    @Tool(description = "Get customer account information including tier, account status, and contact details")
    public CustomerInfo getCustomerInfo(String customerId) {
        return customerService.findById(customerId)
            .map(c -> new CustomerInfo(c.getId(), c.getName(), c.getEmail(),
                                       c.getTier(), c.getStatus()))
            .orElse(null);
    }

    @Tool(description = "Get recent orders for a customer")
    public List<OrderSummary> getRecentOrders(String customerId, int limit) {
        return orderService.findRecentOrders(customerId, limit).stream()
            .map(o -> new OrderSummary(o.getId(), o.getStatus(),
                                       o.getTotalAmount(), o.getCreatedAt().toString()))
            .toList();
    }

    @Tool(description = "Check order status and tracking information")
    public OrderStatus checkOrderStatus(String orderId) {
        return orderService.findById(orderId)
            .map(o -> new OrderStatus(o.getId(), o.getStatus(),
                                      o.getTrackingNumber(), o.getExpectedDelivery()))
            .orElse(null);
    }

    @Tool(description = "Submit a refund request for an order")
    public RefundResult requestRefund(String orderId, String reason) {
        // Validate before submitting
        if (!orderService.isRefundEligible(orderId)) {
            return new RefundResult(false, "Order is not eligible for refund", null);
        }
        String ticketId = orderService.submitRefundRequest(orderId, reason);
        return new RefundResult(true, "Refund request submitted", ticketId);
    }

    @Tool(description = "Create a support ticket for issues that need human attention")
    public TicketResult createSupportTicket(String customerId, String subject,
                                             String description, String priority) {
        String ticketId = customerService.createTicket(customerId, subject, description, priority);
        return new TicketResult(ticketId, "Ticket created. You will hear back within 24 hours.");
    }

    @Tool(description = "Search the knowledge base for help articles")
    public List<String> searchKnowledgeBase(String query) {
        return customerService.searchHelpArticles(query).stream()
            .map(a -> "Title: " + a.title() + "\nURL: " + a.url())
            .toList();
    }

    // Records for tool responses
    record CustomerInfo(String id, String name, String email, String tier, String status) {}
    record OrderSummary(String id, String status, Double amount, String createdAt) {}
    record OrderStatus(String id, String status, String trackingNumber, String expectedDelivery) {}
    record RefundResult(boolean success, String message, String ticketId) {}
    record TicketResult(String ticketId, String message) {}
}
```

```java
// src/main/java/com/example/ai/AgentService.java
package com.example.ai;

import com.example.ai.tools.CustomerTools;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.stereotype.Service;

@Service
public class AgentService {

    private final ChatClient chatClient;

    public AgentService(ChatClient.Builder builder, CustomerTools tools) {
        this.chatClient = builder
            .defaultSystem("""
                You are a helpful customer support agent.
                You have access to tools to look up customer information,
                check orders, and submit requests.
                
                Always verify customer identity before sharing account details.
                Be empathetic and solution-focused.
                If you can't resolve something with the available tools,
                create a support ticket for human follow-up.
                """)
            .defaultTools(tools)   // Register all @Tool methods from the bean
            .build();
    }

    /**
     * Agentic conversation — the AI can call tools multiple times to answer
     */
    public String handleSupportQuery(String customerId, String question) {
        return chatClient.prompt()
            .user(u -> u.text("""
                Customer ID: {customerId}
                Question: {question}
                """)
                .param("customerId", customerId)
                .param("question", question)
            )
            .call()
            .content();
    }
}
```

---

## 10. Conversation Memory {#memory}

```java
// src/main/java/com/example/ai/ConversationalChatService.java
package com.example.ai;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.MessageChatMemoryAdvisor;
import org.springframework.ai.chat.memory.InMemoryChatMemory;
import org.springframework.stereotype.Service;

import java.util.concurrent.ConcurrentHashMap;

@Service
public class ConversationalChatService {

    private final ChatClient chatClient;
    private final ConcurrentHashMap<String, InMemoryChatMemory> sessions = new ConcurrentHashMap<>();

    public ConversationalChatService(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("You are a helpful customer support agent with memory of our conversation.")
            .build();
    }

    /**
     * Chat with per-session memory
     */
    public String chat(String sessionId, String message) {
        InMemoryChatMemory memory = sessions.computeIfAbsent(
            sessionId, k -> new InMemoryChatMemory()
        );

        return chatClient.prompt()
            .advisors(new MessageChatMemoryAdvisor(memory))
            .user(message)
            .call()
            .content();
    }

    /**
     * Clear session memory
     */
    public void clearSession(String sessionId) {
        sessions.remove(sessionId);
    }

    /**
     * Get conversation history
     */
    public int getMessageCount(String sessionId) {
        InMemoryChatMemory memory = sessions.get(sessionId);
        return memory != null ? memory.get(sessionId, Integer.MAX_VALUE).size() : 0;
    }
}
```

---

## 11. Image Generation {#image-gen}

```java
// src/main/java/com/example/ai/ImageGenerationService.java
package com.example.ai;

import org.springframework.ai.image.*;
import org.springframework.ai.openai.OpenAiImageModel;
import org.springframework.ai.openai.OpenAiImageOptions;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class ImageGenerationService {

    private final ImageModel imageModel;

    public ImageGenerationService(ImageModel imageModel) {
        this.imageModel = imageModel;
    }

    /**
     * Generate image from text prompt
     */
    public String generateProductImage(String productDescription) {
        ImageResponse response = imageModel.call(
            new ImagePrompt(
                "Professional product photo of: " + productDescription +
                ". White background, high quality, studio lighting.",
                OpenAiImageOptions.builder()
                    .withModel("dall-e-3")
                    .withQuality("hd")
                    .withN(1)
                    .withHeight(1024)
                    .withWidth(1024)
                    .build()
            )
        );

        return response.getResult().getOutput().getUrl();
    }

    /**
     * Generate multiple variations
     */
    public List<String> generateVariations(String prompt, int count) {
        ImageResponse response = imageModel.call(
            new ImagePrompt(
                prompt,
                OpenAiImageOptions.builder()
                    .withModel("dall-e-2")  // dall-e-2 supports multiple images
                    .withN(count)
                    .withHeight(512)
                    .withWidth(512)
                    .build()
            )
        );

        return response.getResults().stream()
            .map(r -> r.getOutput().getUrl())
            .toList();
    }

    /**
     * Generate with base64 response (for immediate use)
     */
    public byte[] generateAsBytes(String prompt) {
        ImageResponse response = imageModel.call(
            new ImagePrompt(
                prompt,
                OpenAiImageOptions.builder()
                    .withModel("dall-e-3")
                    .withResponseFormat("b64_json")
                    .withHeight(1024)
                    .withWidth(1024)
                    .build()
            )
        );

        String b64 = response.getResult().getOutput().getB64Json();
        return java.util.Base64.getDecoder().decode(b64);
    }
}
```

---

## 12. Cost Management {#cost}

```java
// src/main/java/com/example/ai/cost/UsageTracker.java
package com.example.ai.cost;

import org.springframework.ai.chat.metadata.Usage;
import org.springframework.ai.chat.model.ChatResponse;
import org.springframework.stereotype.Component;

import java.util.concurrent.atomic.AtomicLong;

@Component
public class UsageTracker {

    private final AtomicLong totalInputTokens = new AtomicLong();
    private final AtomicLong totalOutputTokens = new AtomicLong();

    // GPT-4o pricing (as of 2024):
    // Input:  $0.005 per 1K tokens
    // Output: $0.015 per 1K tokens
    private static final double INPUT_COST_PER_1K = 0.005;
    private static final double OUTPUT_COST_PER_1K = 0.015;

    public void track(ChatResponse response) {
        if (response.getMetadata() != null && response.getMetadata().getUsage() != null) {
            Usage usage = response.getMetadata().getUsage();
            totalInputTokens.addAndGet(usage.getPromptTokens());
            totalOutputTokens.addAndGet(usage.getGenerationTokens());
        }
    }

    public double getTotalCostUSD() {
        return (totalInputTokens.get() / 1000.0 * INPUT_COST_PER_1K) +
               (totalOutputTokens.get() / 1000.0 * OUTPUT_COST_PER_1K);
    }

    public UsageSummary getSummary() {
        return new UsageSummary(
            totalInputTokens.get(),
            totalOutputTokens.get(),
            getTotalCostUSD()
        );
    }

    record UsageSummary(long inputTokens, long outputTokens, double costUSD) {}
}
```

```java
// src/main/java/com/example/ai/cost/RateLimitedChatService.java
package com.example.ai.cost;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.stereotype.Service;
import java.time.Duration;
import java.util.concurrent.Semaphore;

@Service
public class RateLimitedChatService {

    private final ChatClient chatClient;
    private final Semaphore semaphore;
    private final UsageTracker usageTracker;

    // Maximum 10 concurrent AI requests
    private static final int MAX_CONCURRENT = 10;

    // Maximum cost per customer per day
    private static final double MAX_DAILY_COST = 5.0;

    public RateLimitedChatService(ChatClient.Builder builder, UsageTracker usageTracker) {
        this.chatClient = builder.build();
        this.usageTracker = usageTracker;
        this.semaphore = new Semaphore(MAX_CONCURRENT);
    }

    public String ask(String question) throws InterruptedException {
        if (usageTracker.getTotalCostUSD() > MAX_DAILY_COST) {
            return "Service is temporarily limited. Please try again later.";
        }

        semaphore.acquire(); // Block if too many concurrent requests
        try {
            var response = chatClient.prompt()
                .user(question)
                .call()
                .chatResponse();

            usageTracker.track(response);
            return response.getResult().getOutput().getContent();
        } finally {
            semaphore.release();
        }
    }
}
```

---

## 13. AI Content Validation {#validation}

```java
// src/main/java/com/example/ai/validation/ContentModerationService.java
package com.example.ai.validation;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.stereotype.Service;

@Service
public class ContentModerationService {

    private final ChatClient moderationClient;

    public ContentModerationService(ChatClient.Builder builder) {
        this.moderationClient = builder
            .defaultSystem("""
                You are a content moderation assistant.
                Analyze text for: harmful content, PII data, profanity, off-topic content.
                Respond ONLY with a JSON object:
                {"safe": true/false, "reasons": ["reason1", ...], "category": "SAFE/PII/HARMFUL/OFFTOPIC"}
                """)
            .build();
    }

    public ModerationResult moderate(String text) {
        ModerationResult result = moderationClient.prompt()
            .user("Analyze: " + text)
            .call()
            .entity(ModerationResult.class);

        return result != null ? result : new ModerationResult(true, java.util.List.of(), "SAFE");
    }

    record ModerationResult(boolean safe, java.util.List<String> reasons, String category) {}
}
```

---

## 14. Real Example: Customer Support Chatbot {#chatbot}

```java
// src/main/java/com/example/ai/chatbot/ChatbotController.java
package com.example.ai.chatbot;

import com.example.ai.tools.CustomerTools;
import com.example.ai.RagService;
import com.example.ai.validation.ContentModerationService;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.MessageChatMemoryAdvisor;
import org.springframework.ai.chat.client.advisor.QuestionAnswerAdvisor;
import org.springframework.ai.chat.memory.InMemoryChatMemory;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;

import java.util.concurrent.ConcurrentHashMap;

@RestController
@RequestMapping("/api/chatbot")
public class ChatbotController {

    private final ChatClient chatClient;
    private final ContentModerationService moderation;
    private final ConcurrentHashMap<String, InMemoryChatMemory> sessions =
        new ConcurrentHashMap<>();

    public ChatbotController(
            ChatClient.Builder builder,
            VectorStore vectorStore,
            CustomerTools customerTools,
            ContentModerationService moderation) {

        this.moderation = moderation;

        this.chatClient = builder
            .defaultSystem("""
                You are Alex, a friendly customer support agent for TechShop.
                
                Your capabilities:
                - Answer questions about products, orders, and policies
                - Look up customer account and order information
                - Process refund requests
                - Create support tickets for complex issues
                
                Guidelines:
                - Be warm, empathetic, and solution-focused
                - Verify customer identity for sensitive operations
                - Keep responses concise (under 150 words unless asked for detail)
                - If unsure, create a ticket rather than guessing
                - Never share other customers' information
                """)
            .defaultAdvisors(
                new QuestionAnswerAdvisor(
                    vectorStore,
                    SearchRequest.defaults().withTopK(3).withSimilarityThreshold(0.65)
                )
            )
            .defaultTools(customerTools)
            .build();
    }

    @PostMapping("/message")
    public ChatResponse chat(@RequestBody ChatRequest request) {
        // Moderate input
        var modResult = moderation.moderate(request.message());
        if (!modResult.safe()) {
            return new ChatResponse(
                "I'm not able to help with that type of request. "
                + "Is there something else I can assist you with?",
                request.sessionId(),
                false
            );
        }

        InMemoryChatMemory memory = sessions.computeIfAbsent(
            request.sessionId(), k -> new InMemoryChatMemory()
        );

        String response = chatClient.prompt()
            .advisors(new MessageChatMemoryAdvisor(memory))
            .user(u -> u.text("""
                Customer ID: {customerId}
                Customer Message: {message}
                """)
                .param("customerId", request.customerId())
                .param("message", request.message())
            )
            .call()
            .content();

        return new ChatResponse(response, request.sessionId(), true);
    }

    @GetMapping(value = "/message/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> chatStream(
            @RequestParam String sessionId,
            @RequestParam String customerId,
            @RequestParam String message) {

        InMemoryChatMemory memory = sessions.computeIfAbsent(
            sessionId, k -> new InMemoryChatMemory()
        );

        return chatClient.prompt()
            .advisors(new MessageChatMemoryAdvisor(memory))
            .user(u -> u.text("Customer ID: {customerId}\nMessage: {message}")
                .param("customerId", customerId)
                .param("message", message)
            )
            .stream()
            .content();
    }

    @DeleteMapping("/session/{sessionId}")
    public void clearSession(@PathVariable String sessionId) {
        sessions.remove(sessionId);
    }

    record ChatRequest(String sessionId, String customerId, String message) {}
    record ChatResponse(String message, String sessionId, boolean success) {}
}
```

```java
// src/main/java/com/example/ai/chatbot/FaqLoader.java
package com.example.ai.chatbot;

import com.example.ai.KnowledgeBaseService;
import com.example.ai.KnowledgeBaseService.FaqEntry;
import jakarta.annotation.PostConstruct;
import org.springframework.stereotype.Component;
import java.util.List;

@Component
public class FaqLoader {

    private final KnowledgeBaseService knowledgeBaseService;

    public FaqLoader(KnowledgeBaseService knowledgeBaseService) {
        this.knowledgeBaseService = knowledgeBaseService;
    }

    @PostConstruct
    public void loadFaqs() {
        knowledgeBaseService.loadFaqDocuments(List.of(
            new FaqEntry(
                "How long does shipping take?",
                "Standard shipping takes 3-5 business days. Express shipping is 1-2 business days. "
                + "Free standard shipping on orders over $50.",
                "shipping"
            ),
            new FaqEntry(
                "What is your return policy?",
                "We accept returns within 30 days of purchase for most items in original condition. "
                + "Electronics have a 15-day return window. "
                + "Start a return at our website or call customer service.",
                "returns"
            ),
            new FaqEntry(
                "How do I track my order?",
                "You'll receive a tracking email once your order ships. "
                + "You can also track orders in your account under Order History. "
                + "Tracking updates within 24 hours of shipping.",
                "orders"
            ),
            new FaqEntry(
                "Are my payment details secure?",
                "Yes! We use PCI-DSS compliant payment processing. "
                + "We never store your full card number. "
                + "All transactions are encrypted with TLS 1.3.",
                "security"
            ),
            new FaqEntry(
                "How do I cancel an order?",
                "Orders can be cancelled within 1 hour of placement. "
                + "After that, you'll need to return the item once delivered. "
                + "Contact support or use the Cancel button in Order History.",
                "orders"
            )
        ));
    }
}
```

---

## Summary

| Feature | API | Notes |
|---|---|---|
| Basic chat | `ChatClient.prompt().user().call()` | One-shot Q&A |
| Streaming | `.stream().content()` returns `Flux<String>` | For UX responsiveness |
| Embeddings | `EmbeddingModel.embed(text)` | Convert text to vector |
| Vector search | `VectorStore.similaritySearch()` | Semantic search |
| RAG | `QuestionAnswerAdvisor` | Auto-retrieval from vector store |
| Tools | `@Tool` annotation | AI calls your Java methods |
| Memory | `MessageChatMemoryAdvisor` | Per-session conversation history |
| Structured output | `.call().entity(MyClass.class)` | Auto-parse JSON to POJO |
| Image generation | `ImageModel.call(ImagePrompt)` | DALL-E, Stable Diffusion |
| Cost tracking | `ChatResponse.getMetadata().getUsage()` | Monitor token spend |

### Best Practices

- Use streaming for all user-facing responses — reduces perceived latency
- Add system prompts to every ChatClient to control behavior
- Store embeddings in PostgreSQL with pgvector for production
- Monitor token usage to avoid surprise bills
- Moderate user inputs before sending to AI
- Implement per-user rate limiting
- Use `QuestionAnswerAdvisor` for RAG — handles embedding and retrieval automatically

---

## Next Part Preview

**Part 076: Advanced Event-Driven Architecture** — process real-time data at scale with Kafka Streams, implementing windowing, joins, state stores, and building a real-time inventory tracking system with exactly-once semantics.
