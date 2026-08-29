---
share_link: https://share.note.sx/ljewc2e5#Er5JXaDyyz7kbvjX1ff01A
share_updated: 2026-08-01T19:22:59+03:00
---
## Table of Contents

1. [[#What Is Spring AI]]
2. [[#Foundations — Core Abstractions]]
3. [[#Your First Spring AI Application]]
4. [[#ChatClient — The Fluent API]]
5. [[#Advisors — Cross-Cutting Concerns]]
6. [[#Tools (Function Calling)]]
7. [[#Embeddings]]
8. [[#Vector Stores]]
9. [[#RAG — Retrieval Augmented Generation]]
10. [[#Evaluation]]
11. [[#Observability]]
12. [[#Guardrails]]
13. [[#Agentic Workflows]]
14. [[#Extra Relevant Points]]
15. [[#Advantages / Disadvantages]]

---

## What Is Spring AI

**Spring AI** — a Spring-ecosystem framework that applies familiar Spring design principles (portability, dependency injection, auto-configuration) to the AI domain: talking to LLMs, generating embeddings, storing/searching vectors, and building RAG/agentic applications — all through **provider-agnostic interfaces**.

```
Your Spring Boot App
        │
        ▼
  Spring AI portable interfaces
  (ChatModel, EmbeddingModel, VectorStore, ImageModel...)
        │
        ▼
  Provider-specific implementation (swappable via config, no code change)
  OpenAI | Anthropic | Ollama | Azure OpenAI | Bedrock | Vertex AI | 20+ others
```

> [!important] Interview trap Q: "If I switch from OpenAI to Anthropic, do I need to rewrite my service code?" A: No — that's the whole point of Spring AI's portability. You depend on the `ChatModel`/`ChatClient` abstraction; swapping providers is mostly a matter of changing the starter dependency and config properties, not your business logic.

---

## Foundations — Core Abstractions

| Abstraction      | Module                 | Role                                                                                                                    |
| ---------------- | ---------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `ChatModel`      | spring-ai-model        | Low-level interface for sending a `Prompt`, getting a `ChatResponse` back from an LLM                                   |
| `ChatClient`     | spring-ai-client-chat  | High-level **fluent** API built on top of `ChatModel` — the one you use day to day                                      |
| `EmbeddingModel` | spring-ai-model        | Converts text into a vector embedding (`float[]`)                                                                       |
| `VectorStore`    | spring-ai-vector-store | CRUD + similarity search over embeddings in a vector database                                                           |
| `Advisor`        | spring-ai-client-chat  | Interceptor-style hook for cross-cutting concerns (memory, RAG, logging, guardrails) applied around a `ChatClient` call |
| `Document`       | spring-ai-model        | Framework's unit of content — text + metadata — used throughout ETL/RAG pipelines                                       |

```
        Prompt (system + user messages)
              │
              ▼
          ChatModel  ──► calls the actual provider API (HTTP)
              │
              ▼
         ChatResponse (AI message + metadata: tokens, finish reason...)
```

---

## Your First Spring AI Application

### 1. Dependency (Spring Boot starter, provider-specific)

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
```

### 2. Configuration

```properties
spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.chat.options.model=gpt-4o
```

### 3. Minimal usage — inject `ChatClient.Builder`, build a `ChatClient`

```java
@RestController
public class ChatController {

    private final ChatClient chatClient;

    public ChatController(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("You are a helpful assistant.")
            .build();
    }

    @GetMapping("/chat")
    public String chat(@RequestParam String message) {
        return chatClient.prompt()
            .user(message)
            .call()
            .content();
    }
}
```

Spring Boot auto-configures a `ChatModel` bean (and thus a `ChatClient.Builder`) from your properties — no manual client construction needed, following the same auto-configuration philosophy as `DataSource` or `RestTemplate`.

> [!tip] Analogy `ChatModel` is like JDBC's raw `Connection` — powerful but low-level. `ChatClient` is like `JdbcTemplate` — the ergonomic, fluent layer most people actually use day to day.

---

## `ChatClient` — The Fluent API

```java
String response = chatClient.prompt()
    .system("Respond like a pirate.")
    .user("Tell me about Java streams.")
    .call()
    .content();
```

|Method|Purpose|
|---|---|
|`.prompt()`|Start building a request|
|`.system(...)` / `.user(...)`|Set system/user message content|
|`.tools(...)`|Attach tools (function calling) for this call|
|`.advisors(...)`|Attach advisors for this call|
|`.call()`|Synchronous execution|
|`.stream()`|Reactive/streaming execution — returns a `Flux<String>`|
|`.content()`|Extract just the text from the response|
|`.chatResponse()`|Get the full `ChatResponse` (metadata, token usage, etc.)|
|`.entity(Class)`|**Structured output** — deserialize the model's response directly into a Java object/record|

### Structured Output

```java
record MovieReview(String title, int rating, String summary) {}

MovieReview review = chatClient.prompt()
    .user("Review the movie Inception in JSON.")
    .call()
    .entity(MovieReview.class);
```

Under the hood, Spring AI generates a JSON-schema instruction appended to the prompt and parses the model's JSON response back into your type — no manual JSON parsing.

---

## Advisors — Cross-Cutting Concerns

**Advisor** — Spring AI's interceptor pattern for `ChatClient` calls, conceptually similar to Spring MVC interceptors or AOP aspects: each advisor can inspect/modify the request before it reaches the model, and inspect/modify the response before it reaches the caller.

```
chatClient.prompt().advisors(A, B, C).user(...).call()

Request flow:   A ──► B ──► C ──► [Model call]
Response flow:  A ◄── B ◄── C ◄── [Model response]

(executes in the order added — put memory advisors before RAG advisors
 if you want memory context to influence retrieval)
```

|Built-in Advisor|Purpose|
|---|---|
|`MessageChatMemoryAdvisor`|Injects prior conversation turns from a `ChatMemory` store|
|`QuestionAnswerAdvisor`|Performs RAG: similarity search + injects retrieved context|
|`RetrievalAugmentationAdvisor`|Modular/configurable RAG pipeline (pre-retrieval, retrieval, post-retrieval stages)|
|`SimpleLoggerAdvisor`|Logs requests/responses for debugging/observability|
|Custom advisors|e.g. `TokenUsageAdvisor`, guardrail advisors, self-correction/retry advisors|

```java
@Bean
ChatClient chatClient(ChatClient.Builder builder, ChatMemory chatMemory, VectorStore vectorStore) {
    return builder
        .defaultAdvisors(
            MessageChatMemoryAdvisor.builder(chatMemory).build(),
            new QuestionAnswerAdvisor(vectorStore),
            new SimpleLoggerAdvisor()
        )
        .build();
}
```

> [!important] Interview trap Q: "What design pattern do Advisors follow?" A: The **chain-of-responsibility / interceptor pattern** — each advisor wraps the next, can short-circuit the chain (e.g. a guardrail blocking a harmful prompt before it reaches the model), and can even **loop back** through the downstream chain (recursive advisors power tool-calling loops, retry-on-invalid-JSON, and self-correction).

---

## Tools (Function Calling)

**Tools** let the model call real application code — databases, APIs, business logic — mid-conversation, rather than being limited to its training data.

```
User: "What's the weather in Cairo, and should I bring an umbrella?"
        │
        ▼
Model decides: "I need the getWeather tool" → emits a tool call request
        │
        ▼
Spring AI's ToolCallingManager EXECUTES the actual Java method
        │
        ▼
Result sent back to the model as a new message
        │
        ▼
Model produces the final natural-language answer using that real data
```

### Declaring a Tool

```java
@Component
class WeatherTools {

    @Tool(description = "Get the current weather for a given city")
    public String getWeather(@ToolParam(description = "City name") String city) {
        return weatherService.fetch(city);
    }
}
```

```java
String answer = chatClient.prompt()
    .user("What's the weather in Cairo?")
    .tools(new WeatherTools())     // scanned via reflection for @Tool methods
    .call()
    .content();
```

Spring AI auto-generates the **JSON Schema** for the tool's parameters from the method signature and sends it to the model as part of the request — the model itself decides _if_ and _when_ to call it.

### Execution Modes

|Mode|Who drives the tool-call loop|
|---|---|
|**Framework-controlled** (default, via `ChatClient`)|Spring AI's `ToolCallingAdvisor` automatically handles the whole loop: call model → detect tool call → execute → send result back → repeat until a final answer|
|**Advisor-controlled**|You customize the loop yourself via a custom advisor|
|**User-controlled**|Full manual control — you inspect tool call requests and decide execution yourself|

> [!important] Interview trap Q: "Does the LLM actually execute your Java method?" A: No — the model only ever _requests_ a tool call (by name + JSON arguments). Spring AI's tool-calling machinery is what actually invokes your real Java method and feeds the result back into the conversation. The model never runs your code directly.

---

## Embeddings

**Embedding** — a numerical vector representation of text (or other content) such that semantically similar content produces vectors that are close together in that vector space.

```
"The cat sat on the mat"  ──EmbeddingModel──►  [0.021, -0.183, 0.774, ..., 0.005]   (e.g. 1536 dimensions)
"A feline rested on the rug"  ──►  [0.019, -0.176, 0.781, ..., 0.011]   ← very close to the vector above!
"Stock market crashed today"  ──►  [0.812, 0.033, -0.291, ..., 0.667]  ← far away
```

```java
EmbeddingModel embeddingModel; // auto-configured bean

float[] vector = embeddingModel.embed("The cat sat on the mat");
```

- The `EmbeddingModel` **only** converts text → vector. It does **not** store or search anything — that's the `VectorStore`'s job.
- Common embedding models: OpenAI `text-embedding-ada-002`/`text-embedding-3-*`, open-source models like Nomic Embed Text (runnable locally via Ollama/LM Studio).

---

## Vector Stores

**`VectorStore`** — Spring AI's portable abstraction for storing embeddings alongside their original text/metadata, and performing **similarity search** against them.

```
vectorStore.add(documents)
        │
        ▼
For each Document: text ──EmbeddingModel──► vector
        │
        ▼
Stored: { id, text, metadata, embedding } in the underlying vector DB

vectorStore.similaritySearch(query)
        │
        ▼
query text ──EmbeddingModel──► query vector
        │
        ▼
DB finds nearest stored vectors (cosine similarity / dot product / etc.)
        │
        ▼
Returns the K most similar original Documents
```

```java
List<Document> docs = List.of(
    new Document("Spring AI simplifies building AI applications on the JVM.")
);
vectorStore.add(docs);   // automatically embeds + stores

List<Document> results = vectorStore.similaritySearch(
    SearchRequest.query("What does Spring AI do?").withTopK(3)
);
```

### Supported Vector Databases (portable — same API, swap the implementation)

|Category|Examples|
|---|---|
|SQL-based|PGVector (Postgres), MongoDB Atlas Vector Search|
|Search engines|Elasticsearch, OpenSearch, Redis|
|Purpose-built vector DBs|Pinecone, Weaviate, Milvus, Qdrant, Chroma|

> [!important] Interview trap Q: "Does `VectorStore.add()` require you to compute embeddings yourself first?" A: No — you just pass in `Document` objects with plain text. `VectorStore` internally calls the configured `EmbeddingModel` for you, then persists both the text and the resulting vector.

---

## RAG — Retrieval Augmented Generation

**RAG** — instead of relying solely on what the model learned during training, you search **your own** knowledge base for relevant content and inject it into the prompt before the model answers — grounding responses in your actual data.

```
1. INGEST (offline, once per document set)
   Raw docs (PDF, text...) → chunk → embed → store in VectorStore

2. QUERY (per user request)
   User question
        │
        ▼
   similaritySearch(question) → top-K relevant chunks
        │
        ▼
   Prompt = system + [retrieved chunks as context] + user question
        │
        ▼
   Model generates an answer GROUNDED in the retrieved context
```

### Ingestion (ETL) Pipeline

```java
PagePdfDocumentReader pdfReader = new PagePdfDocumentReader(resource);
List<Document> chunks = new TokenTextSplitter().apply(pdfReader.read());
vectorStore.add(chunks);
```

|ETL Stage|Component|
|---|---|
|Read|`PagePdfDocumentReader`, `JsonReader`, `TikaDocumentReader`, etc.|
|Split/Chunk|`TokenTextSplitter` and other `DocumentTransformer`s|
|Load|`vectorStore.add(documents)`|

### Simplest RAG — `QuestionAnswerAdvisor`

```java
String answer = chatClient.prompt()
    .advisors(new QuestionAnswerAdvisor(vectorStore))
    .user("What does our refund policy say about late returns?")
    .call()
    .content();
```

The advisor runs a similarity search against `vectorStore`, stuffs the retrieved chunks into the prompt as context, then lets the call proceed as normal.

### Modular RAG — `RetrievalAugmentationAdvisor`

A more configurable, pluggable RAG pipeline (based on the "Modular RAG" architecture), letting you customize each stage:

```java
RetrievalAugmentationAdvisor ragAdvisor = RetrievalAugmentationAdvisor.builder()
    .documentRetriever(VectorStoreDocumentRetriever.builder()
        .vectorStore(vectorStore)
        .similarityThreshold(0.7)
        .build())
    .build();
```

|RAG Stage|What it does|
|---|---|
|Pre-Retrieval|Rewrites/expands the user query to improve search quality|
|Retrieval|Runs the actual similarity search against the `VectorStore`|
|Post-Retrieval|Re-ranks, compresses, or filters retrieved chunks before they hit the prompt|

> [!tip] Analogy RAG is an open-book exam: instead of expecting the model to have memorized everything, you hand it the specific pages (retrieved chunks) relevant to the question right before it answers.

---

## Evaluation

**Evaluation** — automatically judging whether an AI response is actually good (relevant, factually grounded) rather than just trusting that it "looks fine" — critical for RAG pipelines where hallucination is a real risk.

```java
ChatResponse response = chatClient.prompt()
    .advisors(ragAdvisor)
    .user(question)
    .call()
    .chatResponse();

EvaluationRequest evalRequest = new EvaluationRequest(
    question,                                                       // original question
    response.getMetadata().get(RetrievalAugmentationAdvisor.DOCUMENT_CONTEXT), // retrieved context
    response.getResult().getOutput().getText()                      // model's answer
);

EvaluationResponse result = new RelevancyEvaluator(chatClientBuilder).evaluate(evalRequest);
boolean passed = result.isPass();
```

|Evaluator|Checks|
|---|---|
|`RelevancyEvaluator`|Is the answer actually relevant to the user's question (not just factually true but off-topic)?|
|`FactCheckingEvaluator`|Is the answer factually consistent with the retrieved/reference document (catches hallucination)?|

> [!important] Interview trap Q: "Can a factually correct answer still fail evaluation?" A: Yes — `RelevancyEvaluator` specifically checks _relevance_, not just truth. "Lions are the kings of the jungle" is true, but it's a useless (irrelevant) answer to "What is the refund policy?" — that's exactly the failure mode this evaluator is built to catch.

These evaluators are themselves powered by an LLM (an "LLM-as-judge" pattern) — typically wired up as integration tests to catch regressions in prompts, retrieval quality, or model swaps before they reach production.

---

## Observability

Spring AI builds on Spring Boot's existing observability stack (**Micrometer** for metrics, **OpenTelemetry** for tracing) to instrument every core component automatically.

```
ChatClient.call()  ──► spring.ai.chat.client observation (timing + tracing)
      │
      ▼
Advisor chain      ──► spring.ai.advisor observation per advisor
      │
      ▼
ChatModel.call()   ──► gen_ai.client.operation observation
      │
      ▼
VectorStore / EmbeddingModel ──► their own observations too
```

|Instrumented Component|What's Measured|
|---|---|
|`ChatClient` (incl. Advisors)|Time spent per call/advisor, propagated trace context|
|`ChatModel`|Time spent completing the model call|
|`EmbeddingModel`|Embedding generation timing|
|`VectorStore`|Similarity search / add operation timing|

- Prompt/completion content is **not exported by default** (it's often large and can contain sensitive data) — must be explicitly enabled for debugging.
- Token usage is trackable via Micrometer metrics — important because uncontrolled prompt growth (e.g. unbounded chat memory) is a hidden cost driver in production.
- Traces can be exported to tools like Langfuse or any OpenTelemetry-compatible backend to visualize the full request → tool call → response flow.

---

## Guardrails

**Guardrails** — safety/policy checks applied around the model call: blocking harmful or malicious prompts before they reach the model, and sanitizing/redacting sensitive data (e.g. credit card numbers, PII) from responses before users see them.

```
User input ──► [Guardrail Advisor: scan for harmful content] ──► Model
                        │
                        └── if blocked: short-circuit, return a safe refusal, never call the model

Model output ──► [Guardrail Advisor: redact sensitive data] ──► User
```

- Implemented as **Advisors** — the same interceptor mechanism used for memory and RAG — meaning a guardrail can inspect the request pre-call and the response post-call, and can **short-circuit the chain entirely** to block a request without ever hitting the model (saving cost and risk).
- Many model providers (e.g. OpenAI) already include some built-in moderation, but application-level guardrails give you control over domain-specific rules (e.g. stripping PII specific to your business).

```java
class PiiRedactionAdvisor implements CallAdvisor {
    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request, CallAdvisorChain chain) {
        ChatClientResponse response = chain.nextCall(request);
        String redacted = response.chatResponse().getResult().getOutput().getText()
            .replaceAll("\\d{4}-\\d{4}-\\d{4}-\\d{4}", "[REDACTED CARD NUMBER]");
        // return a modified response with redacted text
        return response; // (simplified for illustration)
    }
}
```

---

## Agentic Workflows

**Agentic behavior** — instead of a single prompt → single response, the model plans, calls tools, evaluates results, and iterates across multiple LLM calls until a goal is achieved.

```
Goal: "Book me the cheapest flight from Cairo to Tokyo next month, and reserve a hotel."
        │
        ▼
LLM Call 1: decides to call searchFlights tool
        │
        ▼
Tool executes → results fed back
        │
        ▼
LLM Call 2: decides to call bookFlight, then searchHotels
        │
        ▼
Tool executes → results fed back
        │
        ▼
LLM Call 3: decides to call bookHotel
        │
        ▼
Final natural-language confirmation to the user
```

### Recursive Advisors — The Mechanism Behind Agent Loops

Spring AI's advisor chain supports **looping**: an advisor can re-invoke the downstream chain multiple times, which is exactly how tool-calling loops, retry-on-invalid-structured-output, and self-correcting agents are implemented under the hood.

```java
class SelfCorrectionAdvisor implements CallAdvisor {
    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request, CallAdvisorChain chain) {
        ChatClientResponse response = chain.nextCall(request);
        if (!isValid(response)) {
            ChatClientRequest retryRequest = augmentWithFeedback(request, response);
            // chain.copy(this) re-executes only DOWNSTREAM advisors, avoiding infinite upstream loops
            response = chain.copy(this).nextCall(retryRequest);
        }
        return response;
    }
    @Override public int getOrder() { return 0; } // high priority — wraps other advisors
}
```

|Pattern|How it's realized in Spring AI|
|---|---|
|Tool-calling loop|Built-in `ToolCallingAdvisor`, auto-registered in the advisor chain|
|Structured-output retry|Recursive advisor re-prompts the model if JSON parsing/validation fails|
|Self-correction|Recursive advisor evaluates its own output and retries with feedback|
|Multi-agent orchestration|Multiple `ChatClient`s (each with different tools/system prompts) coordinated by application code or a higher-level advisor|

> [!important] Interview trap Q: "What stops a recursive advisor from looping forever?" A: You must set an explicit **max-iteration limit** — Spring AI doesn't cap this for you by default. Since each iteration is a full LLM call, unbounded loops directly translate into runaway token cost, so limiting attempts and monitoring iteration counts via observability is essential in production.

### MCP (Model Context Protocol)

Spring AI also supports **MCP**, an open protocol for exposing tools/resources to any MCP-compatible AI client (not just Spring AI apps) — letting you build a tool server once and reuse it across different agent frameworks/languages, and secure it with standard mechanisms like OAuth2.

---

## Extra Relevant Points

|Topic|Why it matters|
|---|---|
|**Chat Memory**|`ChatMemory` (e.g. `MessageWindowChatMemory`) persists conversation history across turns via a `ChatMemoryRepository`; without it every call is stateless and the model "forgets" prior turns|
|**Multimodality**|Beyond `ChatModel`, Spring AI also abstracts `ImageModel` (image generation), `TranscriptionModel` (speech-to-text), and multimodal chat (image+text input)|
|**Streaming**|`.stream()` returns a reactive `Flux<String>`, dramatically improving perceived performance for user-facing chat UIs vs waiting for the full response|
|**Prompt templates**|`PromptTemplate` supports parameterized prompts (`{query}`, `{target}`) rather than raw string concatenation — safer and more maintainable|
|**Auto-configuration**|Following core Spring Boot philosophy — beans like `ChatModel`, `EmbeddingModel`, `VectorStore` are wired automatically from `application.properties`, no manual bean construction for the common case|
|**Testing**|Evaluators (`RelevancyEvaluator`, `FactCheckingEvaluator`) are designed to be used as JUnit integration tests, catching prompt/retrieval regressions in CI, not just at runtime|
|**Cost control**|Token usage should be actively monitored (Micrometer) — unbounded chat memory or unchecked agentic loops are common hidden cost drivers|

---

## Advantages / Disadvantages

|Advantages|Disadvantages|
|---|---|
|Provider-agnostic — swap OpenAI/Anthropic/Ollama/etc. with config, not code|Rapidly evolving API surface (breaking changes between versions are common)|
|Deep Spring Boot integration — auto-configuration, familiar DI patterns|Additional abstraction layer can obscure provider-specific features/quirks|
|Built-in RAG, tool calling, memory, evaluation, observability — less boilerplate|LLM-as-judge evaluators add latency/cost to test suites|
|Advisor pattern makes cross-cutting concerns composable and testable|Agentic/recursive workflows can produce runaway token costs if not capped|
|First-class Micrometer/OpenTelemetry observability|Still maturing compared to older, more established Spring modules|