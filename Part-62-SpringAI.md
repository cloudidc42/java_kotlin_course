# Part 62: Spring AI & LLM Integration
## ขั้นตอนที่ 4241-4310: Integrating AI/LLM into Java/Kotlin Applications

---

## 62.1 Spring AI Overview

```
Spring AI คืออะไร:
  - Spring framework สำหรับสร้าง AI-powered applications
  - รองรับ OpenAI, Anthropic Claude, Google Gemini, Ollama (local)
  - Abstraction layer: เปลี่ยน model provider โดยไม่ต้องแก้ business code
  - Features: Chat, Embedding, Image generation, Function calling

Use Cases:
  ✓ Chatbot สำหรับ customer service
  ✓ Code generation / review
  ✓ Document summarization / Q&A
  ✓ Sentiment analysis
  ✓ Product recommendation
  ✓ Intelligent search (RAG: Retrieval Augmented Generation)
```

---

## 62.2 Setup & Configuration

```xml
<!-- pom.xml -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- OpenAI -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    </dependency>
    
    <!-- Anthropic Claude -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-anthropic-spring-boot-starter</artifactId>
    </dependency>
    
    <!-- Vector store (PGVector) -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o
          temperature: 0.7
          max-tokens: 2000
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-opus-4-5
          temperature: 0.5
```

---

## 62.3 Basic Chat Integration

```java
import org.springframework.ai.chat.client.*;
import org.springframework.ai.chat.messages.*;
import org.springframework.ai.chat.model.*;
import org.springframework.stereotype.*;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/ai")
class ChatController {
    
    private final ChatClient chatClient;
    
    ChatController(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("คุณเป็น assistant ที่ช่วยตอบคำถามเกี่ยวกับสินค้าของเรา ตอบเป็นภาษาไทย")
            .build();
    }
    
    // Simple chat
    @PostMapping("/chat")
    String chat(@RequestBody ChatRequest request) {
        return chatClient.prompt()
            .user(request.message())
            .call()
            .content();
    }
    
    // Streaming response (Server-Sent Events)
    @GetMapping(value = "/chat/stream", produces = "text/event-stream")
    reactor.core.publisher.Flux<String> chatStream(@RequestParam String message) {
        return chatClient.prompt()
            .user(message)
            .stream()
            .content();
    }
    
    // Chat with history (multi-turn conversation)
    @PostMapping("/chat/session")
    String chatWithHistory(@RequestBody ChatSessionRequest request) {
        var messages = new java.util.ArrayList<Message>();
        
        // Add conversation history
        for (var msg : request.history()) {
            if ("user".equals(msg.role())) {
                messages.add(new UserMessage(msg.content()));
            } else {
                messages.add(new AssistantMessage(msg.content()));
            }
        }
        
        // Add current message
        messages.add(new UserMessage(request.message()));
        
        return chatClient.prompt()
            .messages(messages)
            .call()
            .content();
    }
    
    record ChatRequest(String message) {}
    record ChatMessage(String role, String content) {}
    record ChatSessionRequest(String message, java.util.List<ChatMessage> history) {}
}
```

---

## 62.4 Prompt Templates

```java
import org.springframework.ai.chat.prompt.*;
import org.springframework.core.io.*;

@Service
class ProductAssistantService {
    
    private final ChatClient chatClient;
    
    ProductAssistantService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }
    
    // Prompt template with variables
    public String describeProduct(String productName, String category, double price) {
        return chatClient.prompt()
            .system("""
                คุณเป็นนักเขียนคำโฆษณาที่เชี่ยวชาญ
                สร้างคำอธิบายสินค้าที่น่าดึงดูดใจ กระชับ และตรงกลุ่มลูกค้า
                """)
            .user(u -> u.text("""
                สร้างคำอธิบายสินค้าสำหรับ:
                ชื่อ: {name}
                หมวดหมู่: {category}
                ราคา: {price} บาท
                
                ใช้ประโยคไม่เกิน 3 ประโยค เน้นจุดขาย
                """)
                .param("name", productName)
                .param("category", category)
                .param("price", price)
            )
            .call()
            .content();
    }
    
    // Structured output (AI returns JSON mapped to Java class)
    public ProductAnalysis analyzeProduct(String productDescription) {
        return chatClient.prompt()
            .user("""
                วิเคราะห์สินค้าต่อไปนี้และตอบกลับเป็น JSON:
                %s
                """.formatted(productDescription))
            .call()
            .entity(ProductAnalysis.class);  // Spring AI maps response to class
    }
    
    // Function calling: AI can call our Java functions
    public String answerWithData(String question) {
        return chatClient.prompt()
            .user(question)
            .functions("getProductPrice", "checkInventory")  // AI will call these when needed
            .call()
            .content();
    }
    
    record ProductAnalysis(
        String name,
        String category,
        String targetAudience,
        java.util.List<String> keyFeatures,
        String sentiment
    ) {}
}
```

---

## 62.5 RAG (Retrieval Augmented Generation)

```java
import org.springframework.ai.vectorstore.*;
import org.springframework.ai.document.*;
import org.springframework.ai.embedding.*;
import org.springframework.ai.reader.*;

// RAG Architecture:
// 1. Ingest: Load documents → Split → Embed → Store in vector DB
// 2. Query: User question → Embed → Find similar docs → LLM answers with context

@Service
class KnowledgeBaseService {
    
    private final VectorStore vectorStore;
    private final ChatClient chatClient;
    
    KnowledgeBaseService(VectorStore vectorStore, ChatClient.Builder builder) {
        this.vectorStore = vectorStore;
        this.chatClient = builder.build();
    }
    
    // ====== Ingest documents into vector store ======
    public void ingestDocuments(java.util.List<String> texts, String source) {
        var documents = texts.stream()
            .map(text -> new Document(text, java.util.Map.of("source", source)))
            .toList();
        
        // Automatically embeds and stores in vector DB
        vectorStore.add(documents);
        System.out.println("Ingested " + documents.size() + " documents from " + source);
    }
    
    // Ingest from file
    public void ingestFile(org.springframework.core.io.Resource resource) {
        var reader = new org.springframework.ai.reader.TextReader(resource);
        var splitter = new org.springframework.ai.transformer.splitter.TokenTextSplitter();
        
        var documents = splitter.apply(reader.get());
        vectorStore.add(documents);
    }
    
    // ====== Answer question using knowledge base ======
    public String answerQuestion(String question) {
        // 1. Find relevant documents
        var relevantDocs = vectorStore.similaritySearch(
            SearchRequest.query(question)
                .withTopK(5)
                .withSimilarityThreshold(0.7)
        );
        
        if (relevantDocs.isEmpty()) {
            return "ขออภัย ไม่พบข้อมูลที่เกี่ยวข้องกับคำถามนี้";
        }
        
        // 2. Build context from retrieved documents
        var context = relevantDocs.stream()
            .map(Document::getContent)
            .collect(java.util.stream.Collectors.joining("\n\n---\n\n"));
        
        // 3. Ask LLM with context
        return chatClient.prompt()
            .system("""
                ตอบคำถามโดยอ้างอิงจากข้อมูลที่ให้มาเท่านั้น
                ถ้าไม่มีข้อมูลเพียงพอ ให้บอกว่าไม่ทราบ
                ตอบเป็นภาษาไทย
                """)
            .user("""
                ข้อมูลอ้างอิง:
                {context}
                
                คำถาม: {question}
                """)
            .call()
            .content()
            // Manually substitute (or use param())
            .replace("{context}", context)
            .replace("{question}", question);
    }
    
    // Answer with source attribution
    public QuestionAnswer answerWithSources(String question) {
        var relevantDocs = vectorStore.similaritySearch(
            SearchRequest.query(question).withTopK(3)
        );
        
        var context = relevantDocs.stream()
            .map(Document::getContent)
            .collect(java.util.stream.Collectors.joining("\n\n"));
        
        var sources = relevantDocs.stream()
            .map(doc -> (String) doc.getMetadata().get("source"))
            .filter(s -> s != null)
            .distinct()
            .toList();
        
        var answer = chatClient.prompt()
            .user("Context:\n" + context + "\n\nQuestion: " + question)
            .call()
            .content();
        
        return new QuestionAnswer(question, answer, sources);
    }
    
    record QuestionAnswer(String question, String answer, java.util.List<String> sources) {}
}

// PGVector configuration
@org.springframework.context.annotation.Configuration
class VectorStoreConfig {
    
    @org.springframework.context.annotation.Bean
    VectorStore vectorStore(
            org.springframework.ai.embedding.EmbeddingModel embeddingModel,
            javax.sql.DataSource dataSource) {
        
        return org.springframework.ai.vectorstore.pgvector.PgVectorStore.builder(
                new org.springframework.jdbc.core.JdbcTemplate(dataSource),
                embeddingModel)
            .dimensions(1536)  // OpenAI text-embedding-3-small
            .distanceType(org.springframework.ai.vectorstore.pgvector.PgVectorStore.PgDistanceType.COSINE_DISTANCE)
            .initializeSchema(true)
            .build();
    }
}
```

---

## 62.6 Function Calling (Tool Use)

```java
import org.springframework.ai.model.function.*;
import org.springframework.context.annotation.*;

// Function calling: LLM decides when to call our functions

@Configuration
class AiFunctionConfig {
    
    // Define function that AI can call
    @Bean
    @Description("Get current price and stock status of a product by ID")
    java.util.function.Function<ProductPriceRequest, ProductPriceResponse> getProductPrice(
            ProductRepository productRepository) {
        
        return request -> {
            var product = productRepository.findById(request.productId())
                .orElseThrow(() -> new RuntimeException("Product not found: " + request.productId()));
            
            return new ProductPriceResponse(
                product.getId(),
                product.getName(),
                product.getPrice(),
                product.getStockQuantity() > 0 ? "In Stock" : "Out of Stock"
            );
        };
    }
    
    @Bean
    @Description("Get current weather for a city (returns temperature in Celsius)")
    java.util.function.Function<WeatherRequest, WeatherResponse> getCurrentWeather() {
        return request -> {
            // In real app: call weather API
            return new WeatherResponse(request.city(), 28.5, "Sunny", 65);
        };
    }
    
    record ProductPriceRequest(String productId) {}
    record ProductPriceResponse(String id, String name, double price, String status) {}
    record WeatherRequest(String city) {}
    record WeatherResponse(String city, double temperature, String condition, int humidity) {}
}

@RestController
class SmartAssistantController {
    
    private final ChatClient chatClient;
    
    SmartAssistantController(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultFunctions("getProductPrice", "getCurrentWeather")
            .build();
    }
    
    @PostMapping("/api/ai/smart-chat")
    String smartChat(@RequestBody String question) {
        // AI will automatically call getProductPrice or getCurrentWeather when needed
        return chatClient.prompt()
            .user(question)
            .call()
            .content();
    }
}
// Example: "ราคาของสินค้า P001 เป็นเท่าไหร่?"
// → AI calls getProductPrice("P001") → gets data → answers user
```

---

## สรุป Part 62

```
Spring AI Key Concepts:

Chat Models:
  ChatClient.Builder → ChatClient → .prompt().user(...).call().content()
  Streaming: .stream().content() → Flux<String> (SSE)
  
Prompt Engineering:
  .defaultSystem() = system prompt สำหรับทุก request
  .user(u -> u.text("...").param("key", value)) = template with variables
  .entity(SomeClass.class) = structured output (JSON → Java object)

RAG (Retrieval Augmented Generation):
  1. Ingest: Document → Embed → VectorStore
  2. Query: Question → Embed → Similarity Search → Context → LLM
  Use: PGVector, Pinecone, Chroma, Redis as vector store
  
Function Calling:
  @Bean @Description("...") Function<Req, Res> myFunction()
  chatClient.defaultFunctions("myFunction")
  AI decides when to call → gets result → answers user

Supported Models:
  OpenAI: gpt-4o, gpt-4o-mini
  Anthropic: claude-opus-4-5, claude-sonnet-4-5
  Google: gemini-1.5-pro
  Ollama: llama3, mistral (local, no API cost)
```

➡️ [Part 63: Elasticsearch & Full-Text Search](./Part-63-Elasticsearch.md)
