---
share_link: https://share.note.sx/b14ybtau#qPyjtRy7G5aEfqFVkW5bYQ
share_updated: 2026-08-01T19:22:54+03:00
---
## Part 1: AI Basics (Generative AI Fundamentals)

**1. What is an LLM (Large Language Model)?** A machine learning model trained on massive amounts of text to predict the next piece of text given some input. It's the underlying engine behind tools like ChatGPT, Claude, or Gemini — used for tasks like answering questions, summarizing, or generating code.

**2. What is a token, and why does it matter?** A token is the smallest unit of text an AI model processes — this can be a whole word, part of a word, or punctuation. Models have limits on how many tokens they can handle in one request (context window), and API costs are usually billed per token, so it directly affects both cost and how much context you can send.

**3. What is "temperature" in the context of an AI model?** A parameter that controls randomness/creativity in the output. A low temperature (e.g., 0.1–0.3) makes responses more deterministic and focused — good for factual tasks. A high temperature (e.g., 0.7–1.0) makes responses more varied and creative — good for brainstorming or content generation.

**4. What is a prompt?** The input text sent to an AI model to guide its response. It can be a simple question or a detailed instruction with context, constraints, and formatting requirements.

**5. What's a system prompt vs. a user prompt?**

- **System prompt**: sets the model's role, behavior, and constraints for the whole conversation (e.g., "You are a helpful customer support assistant, only answer questions about our product").
- **User prompt**: the actual question or request from the end user.

**6. What is prompt engineering?** The practice of designing and refining prompts to get more accurate, consistent, and useful responses from an AI model — e.g., giving the model a role, providing context, specifying an output format, and adding constraints.

**7. What is context window / context length?** The maximum amount of text (measured in tokens) a model can "see" at once, including both the prompt and its own response. If a conversation or document exceeds it, older content gets truncated or must be summarized/managed manually.

**8. What is an embedding?** A numerical vector representation of text (or images) that captures its semantic meaning. Texts with similar meaning end up with vectors that are close together in the embedding space — this is the foundation for semantic search and RAG.

**9. What's the difference between a Chat model and a Completion model?**

- **Completion models** take a single block of text and continue it.
- **Chat models** are structured around a conversation made up of messages with roles (system, user, assistant) — the standard interface for most modern LLM APIs today, including Spring AI's `ChatModel`.

**10. What is "hallucination" in AI?** When a model generates information that sounds plausible and confident but is factually incorrect or made up — a key risk to manage in production AI applications, often mitigated through RAG, grounding in real data, and validation.

**11. What is fine-tuning, and how is it different from prompting?**

- **Prompting**: guiding a general-purpose pretrained model's behavior at request time, without changing the model itself.
- **Fine-tuning**: further training a base model on your own specific dataset so it permanently "learns" a style, domain, or task — more expensive and less flexible than prompting, but useful for specialized behavior.

**12. What is function calling / tool calling in the context of LLMs?** A capability where the model can decide to invoke a predefined function/tool (e.g., "get current weather", "look up an order by ID") instead of just answering in plain text — the model outputs which function to call and with what arguments, and the application executes it and feeds the result back. This lets AI applications take real actions or fetch live data.

**13. What is a vector database, and why is it used with AI applications?** A database optimized for storing and searching high-dimensional vectors (embeddings), using similarity search (e.g., cosine similarity) instead of exact matches. It's the backbone of semantic search and RAG systems, since it lets you find "meaning-similar" content rather than just keyword matches. Examples: Pinecone, PGVector, Redis, Milvus, Chroma.

---

## Part 2: RAG (Retrieval-Augmented Generation)

**14. What is RAG, and why is it needed?** RAG (Retrieval-Augmented Generation) is a pattern where, instead of relying purely on what the LLM learned during training, you **retrieve relevant information from an external knowledge source** (like your own documents or database) and inject it into the prompt before the model generates a response. It's needed because LLMs:

- Have a training cutoff (they don't know about your private data or recent events)
- Can hallucinate when they don't actually know something
- Can't be retrained cheaply every time your data changes

RAG lets you ground the model's answers in accurate, up-to-date, domain-specific information without retraining it.

**15. What are the main steps in a typical RAG pipeline?**

1. **Ingestion**: load your documents (PDFs, docs, DB records, etc.)
2. **Chunking**: split documents into smaller pieces
3. **Embedding**: convert each chunk into a vector using an embedding model
4. **Storage**: store those vectors in a vector database
5. **Retrieval**: at query time, embed the user's question and find the most similar chunks in the vector store
6. **Augmentation**: insert the retrieved chunks into the prompt as context
7. **Generation**: send the augmented prompt to the LLM to generate a grounded answer

**16. Why is chunking important, and what factors affect chunk size?** Documents are usually too long to embed and retrieve as a single unit, so they're split into smaller "chunks" (e.g., paragraphs). Chunk size matters because:

- Too large → chunks may cover multiple unrelated topics, reducing retrieval precision, and may exceed context limits
- Too small → chunks may lose important surrounding context

Common strategies: fixed token/character size with overlap, or splitting by semantic boundaries (paragraphs, sections).

**17. What is semantic search, and how is it different from keyword search?**

- **Keyword search** matches exact words/phrases (like a traditional SQL `LIKE` query or full-text search).
- **Semantic search** compares the _meaning_ of the query and documents using embeddings/vector similarity — so a search for "car" can match a document about "automobile" even without the exact word.

**18. What is a similarity metric, and which ones are commonly used?** A way to measure how "close" two embedding vectors are. Common metrics: **cosine similarity** (most common — measures the angle between vectors), Euclidean distance, and dot product.

**19. What's the difference between "retrieval" and "generation" in RAG?**

- **Retrieval**: the search step — finding the most relevant chunks of information from your data source.
- **Generation**: the LLM step — using the retrieved chunks plus the user's question to produce a natural-language answer.

**20. What are common problems/challenges with RAG systems?**

- Poor chunking leading to irrelevant or incomplete context
- Retrieval returning documents that aren't actually relevant (bad embeddings or query mismatch)
- Context window limits — can't stuff in too many retrieved chunks
- Outdated or duplicate content in the knowledge base
- The model still hallucinating even with correct context if the prompt isn't structured well

**21. What is metadata filtering in a RAG system?** Combining vector similarity search with traditional filters (like SQL `WHERE` clauses) — e.g., "find the most similar chunks, but only from documents tagged `category: HR` and `year: 2026`." This narrows retrieval to relevant subsets of data before or alongside the similarity search.

**22. What's the difference between RAG and fine-tuning — when would you choose one over the other?**

- **RAG**: best when your knowledge base changes often, you need traceability/grounding (can cite sources), and you don't want the cost/complexity of retraining.
- **Fine-tuning**: best when you need the model to adopt a specific tone/style/format consistently, or perform a very specialized task, and your "knowledge" doesn't change often. They're not mutually exclusive — some systems use both.

**23. What is re-ranking in a RAG pipeline?** An optional extra step after initial retrieval where a more precise (but often more expensive) model re-scores and reorders the top retrieved chunks by actual relevance to the query, improving accuracy before passing the final set to the LLM.

---

## Part 3: Spring AI

**24. What is Spring AI?** A Spring project that provides a standardized, Spring-friendly abstraction layer for integrating AI capabilities (chat, embeddings, image generation, etc.) into Java/Spring Boot applications — so you can work with different AI providers (OpenAI, Azure OpenAI, Anthropic, Google Vertex AI, Ollama, etc.) through a consistent API, similar to how Spring Data abstracts different databases.

**25. Why use Spring AI instead of calling an AI provider's REST API directly?**

- Provides a **unified API** — swap providers (e.g., OpenAI → Anthropic) with minimal code changes
- Integrates naturally with Spring Boot's auto-configuration, dependency injection, and starters
- Handles boilerplate like building requests, parsing responses, and mapping structured output to POJOs
- Comes with built-in support for RAG components: vector stores, document readers/splitters, and embedding models
- Adds observability hooks for monitoring AI calls like any other Spring-managed component

**26. What are the two main client APIs in Spring AI, and how do they differ?**

- **`ChatModel`** — the lower-level interface for sending a `Prompt` and getting back a `ChatResponse`. Gives fine-grained control.
- **`ChatClient`** — a higher-level, fluent builder API (similar in spirit to `WebClient`) for constructing chat interactions more concisely, including easier support for prompt templates, structured output conversion, and chaining calls with advisors (like memory or RAG).

**27. What is the `Prompt` class in Spring AI?** It represents the full input sent to a chat model — a list of `Message` objects (system, user, assistant messages) plus optional `ChatOptions` (like temperature, max tokens, model name).

**28. What are the different `Message` types in Spring AI, and what's each for?**

- **SystemMessage** — sets the model's role/behavior for the conversation
- **UserMessage** — represents input from the end user
- **AssistantMessage** — represents a previous response from the AI (used to maintain conversation history)
- Some providers also support a **ToolMessage**-type role for returning function/tool call results back to the model

**29. What is `ChatOptions` used for?** An interface for configuring model-specific parameters like `temperature`, `maxTokens`, `topP`, `topK`, and `model` name — each provider (e.g., `OpenAiChatOptions`) has its own implementation exposing provider-specific settings while still fitting the common interface.

**30. How does Spring AI support streaming responses?** Through the `StreamingChatModel` interface, which returns a `Flux<ChatResponse>` (using Project Reactor) instead of a single blocking response — useful for showing tokens as they're generated, like a typing chat UI.

**31. How would you configure Spring AI to use OpenAI in `application.properties`?**

```properties
spring.ai.openai.api-key=your_api_key_here
spring.ai.openai.chat.options.model=gpt-4o
```

Spring Boot's auto-configuration then wires up a `ChatModel` bean automatically once the relevant starter dependency (e.g., `spring-ai-openai-spring-boot-starter`) is on the classpath — no manual bean configuration needed for the basic case.

**32. What is the Spring AI BOM (Bill of Materials), and why is it used?** A `pom.xml` entry (`spring-ai-bom`) that centrally manages the versions of all Spring AI-related dependencies, so you don't need to specify a version on every individual Spring AI starter — it keeps versions consistent and avoids conflicts across the project.

**33. How does Spring AI support Retrieval-Augmented Generation (RAG)?** Spring AI provides abstractions for the whole RAG pipeline:

- **DocumentReader** — loads content from sources (PDF, text, JSON, etc.)
- **DocumentTransformer / TextSplitter** — chunks documents into smaller pieces
- **EmbeddingModel** — converts text into vector embeddings
- **VectorStore** — a common interface for storing/searching embeddings, with implementations for providers like PGVector, Redis, Pinecone, MongoDB Atlas, Cassandra, and more
- **Advisors** (e.g., `QuestionAnswerAdvisor`) — can be attached to `ChatClient` to automatically retrieve relevant context from a `VectorStore` and inject it into the prompt before calling the model

**34. What is the `VectorStore` interface in Spring AI?** A common abstraction for storing and querying embeddings, regardless of the underlying vector database. It exposes methods to add documents (auto-embedding them) and to perform similarity search, so switching vector database providers mostly means changing configuration/dependency rather than application code.

**35. What is an "Advisor" in Spring AI (e.g., in `ChatClient`)?** A pluggable interceptor-like component that wraps around a chat interaction to add cross-cutting behavior — for example, `QuestionAnswerAdvisor` automatically performs RAG retrieval and injects context, while other advisors can add conversational memory (remembering prior turns) or logging. Conceptually similar to a Spring interceptor/filter, but for AI calls.

**36. How does Spring AI handle structured output (mapping AI responses to Java objects)?** Through output converters (e.g., `BeanOutputConverter`), which instruct the model (via the prompt) to respond in a specific JSON structure, then automatically deserialize that JSON into a POJO — so instead of manually parsing text responses, you can get typed Java objects directly back from the model.

**37. What is Function Calling / Tool support in Spring AI?** Spring AI lets you register Java methods (annotated or registered as `@Tool`/`FunctionCallback`, depending on version) as callable "tools" the AI model can invoke — e.g., a `getWeather(city)` method. When the model decides it needs external/live data, it requests a call to that function with the right arguments, Spring AI executes the actual Java method, and the result is fed back into the conversation so the model can produce a final grounded answer.

**38. What vector store implementations does Spring AI support?** Multiple, via separate starters — common ones include PostgreSQL/PGVector, Redis, Pinecone, MongoDB Atlas, Cassandra, Elasticsearch, Milvus, and Chroma. You choose one by adding the corresponding Spring AI starter and configuring connection properties; application code interacting with `VectorStore` mostly stays the same across providers.

**39. How would you keep conversation history/memory across multiple chat turns in Spring AI?** Spring AI supports conversational memory (often via a memory-related `Advisor` or explicit message list management) so previous `UserMessage`/`AssistantMessage` pairs are included in subsequent prompts — this gives the model context of the ongoing conversation instead of treating every request as brand new.

**40. What's the role of `EmbeddingModel` in Spring AI?** An interface abstracting different embedding providers (OpenAI embeddings, etc.) — used to convert text into vector representations, which are then stored in a `VectorStore` for later similarity search (the core mechanism behind RAG retrieval).

**41. How would you test a Spring AI-powered service without calling a real, paid AI API in your tests?** Similar to testing any external dependency: mock the `ChatModel` (or `ChatClient`) bean with `@MockBean`/Mockito and stub its response, so your Service/Controller logic is tested in isolation without making real API calls or incurring costs.

**42. What are typical use cases for Spring AI in a backend application?**

- AI-powered chatbots / customer support assistants
- Document summarization tools
- Semantic search over internal knowledge bases (RAG)
- Code review or content generation assistants
- Structured data extraction from unstructured text (e.g., extracting fields from a resume or invoice into a POJO)

---
## Vector Search with Relational DBs & Hibernate

**43. Vector search with a relational DB — is it possible?**
Yes. PostgreSQL is the main relational DB with real vector support via the **pgvector** extension, which adds a native `vector` column type plus distance operators (`<->` for L2, `<#>` for inner product, `<=>` for cosine) and specialized indexes. Other relational engines are adding similar capabilities (SQL Server, Oracle 23ai have vector types too), but pgvector is the most mature/common option in the Java ecosystem. This means you get vector similarity search **inside** your normal relational schema — no separate vector database needed for many use cases.

**44. How to embed a new type (i.e., pgvector's `vector`) to be used by Hibernate**
There are two eras here:

- **Before Hibernate 6.4**: no built-in support. You had to write a custom Hibernate `UserType` yourself that knew how to serialize a `float[]`/`List<Float>` to pgvector's wire format and back.
- **Hibernate 6.4+**: there's now an official `hibernate-vector` module that handles this natively.

```java
@Entity
class Item {
    @Id @GeneratedValue
    private Long id;

    @JdbcTypeCode(SqlTypes.VECTOR)
    @Array(length = 384) // dimensions
    private float[] embedding;
}
```

Add the dependency:

```xml
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-vector</artifactId>
    <version>6.4.0.Final</version> <!-- match your Hibernate version -->
</dependency>
```

You can then query with distance functions directly in HQL:

```java
entityManager.createQuery(
    "FROM Item ORDER BY l2_distance(embedding, :embedding) LIMIT 5", Item.class)
    .setParameter("embedding", new float[]{1, 1, 1})
    .getResultList();
```

This is now a first-class, officially supported mapping path — not a hack.

**45. How does Hibernate handle unknown/unsupported types?**
Hibernate maps Java types to JDBC types via its **type system** (`BasicType`, `JdbcType`, `JavaType` in modern Hibernate 6). For anything it doesn't recognize out of the box, there are a few escape hatches:

- **`@JdbcTypeCode(SqlTypes.X)`** — tell Hibernate to treat the column as a specific SQL type code. This is exactly what the vector module uses (`SqlTypes.VECTOR`).
- **A custom `UserType`** — you implement the conversion logic between your Java type and the JDBC representation yourself.
- **`@Convert` / `AttributeConverter`** (standard JPA) — a simpler converter interface, good for many custom-mapping cases, though less flexible than a full `UserType` for exotic column types like arrays/vectors.
- **Custom `Dialect` extensions** — needed if the database dialect itself must understand a new SQL type, which is effectively what `hibernate-vector` bundles for PostgreSQL.

If Hibernate truly has no idea what to do with a type, it fails at mapping/schema-generation time, or throws at runtime when trying to bind/read the column.

**46. Can you introduce a new type to Hibernate? (Yes, but not recommended)**
Yes — via `UserType`, `AttributeConverter`, or `@JdbcTypeCode`, as above. It's a well-supported extension point, not a hack.

**Why it's "not recommended" as a general rule:** for most custom types, it adds real maintenance burden — you're taking on hand-written serialization/deserialization logic, and it can behave inconsistently across Hibernate versions or between DB dialects if not carefully written (e.g., something that works in native SQL queries but breaks in HQL, or works for reads but not schema generation). It also tends to leak database-specific concepts into your entity model, which undermines the ORM's goal of abstracting the database away.

So the practical guidance is: "yes you can, but only do it when there's no first-class or maintained-library option available." This is exactly why the existence of the official `hibernate-vector` module matters — it means you no longer _need_ to write a custom type for this specific case.

**46. Can you use Hibernate with a dedicated vector DB (Quadrant, Pinecone, Milvus, Weaviate, etc.)?**
**No, not meaningfully.** Hibernate is a relational/JPA ORM — it fundamentally assumes SQL, tables, columns, transactions, and a JDBC driver. Dedicated vector databases are usually **not relational** and don't expose a JDBC interface at all (Pinecone, Milvus, Weaviate, Chroma all have their own client SDKs/gRPC/REST APIs, not SQL). There's no JDBC driver to plug into Hibernate for these, so there's no dialect Hibernate could target.

If the goal is "Hibernate + vector DB" for a proof of concept, what people usually actually mean is one of:

1. **Using pgvector inside Postgres** — this is genuinely Hibernate + vectors, as described above. Most likely what you actually want for a class project.
2. **Using Hibernate for relational data (users, orders, metadata) and separately using a vector DB's own SDK for the vector part**, with your application code as the glue. Hibernate doesn't have to be your _whole_ persistence layer — this is common in real systems.
3. **Building a custom Hibernate dialect/driver that fakes a JDBC-like interface over a vector DB** — theoretically possible as a novelty/proof-of-concept, but nobody does this in practice, since it fights against what both tools are actually good at.

For an honest, correct proof-of-concept architecture: use option 2 — Hibernate handles normal entities, and you call the vector DB's own client separately for similarity search, then join results back to relational data by ID in application code.

**47. When to use embeddings, and when not to?**

**Use embeddings when:**

- You need **semantic/similarity** matching — "find things that mean something similar" rather than exact keyword matches (recommendations, semantic search, RAG, deduplication/clustering, anomaly detection).
- The data is unstructured or fuzzy (text, images, audio) where exact filters don't capture what "similar" means.
- You want to match across paraphrases, synonyms, or cross-lingual similarity.

**Don't use embeddings when:**

- You need **exact match or structured filtering** — e.g., "find orders where status = 'SHIPPED' and amount > 100" is a plain SQL query; forcing this through vector similarity is slower and less precise than just indexing the column normally.
- The data volume is tiny, or a simple full-text search (`LIKE`, PostgreSQL's built-in `tsvector`/full-text search) already solves the problem — embeddings add infrastructure and cost for no real benefit.
- You need guaranteed, deterministic, explainable results — similarity search returns "close enough" matches ranked by distance, not a provably correct answer, which can be a poor fit for compliance/audit-sensitive lookups.
- Latency budgets are extremely tight and the vector index/search adds unacceptable overhead compared to a simple indexed query.

**48. What are the trade-offs of using embeddings?**

|Aspect|Trade-off|
|---|---|
|**Accuracy vs. cost**|Semantic search finds conceptually related results traditional search misses, but it costs compute to generate embeddings (calling an embedding model) and storage for the vectors themselves.|
|**Precision vs. recall**|Vector similarity is approximate by nature — you get "close" results, not exact ones. Great for fuzzy matching, bad when you need a guaranteed correct/exact answer.|
|**Freshness**|If underlying data changes, embeddings must be regenerated and re-indexed — this isn't automatic, and stale embeddings silently degrade result quality without throwing errors.|
|**Explainability**|Harder to explain _why_ two things were considered similar (a distance score isn't intuitive to a non-technical stakeholder), compared to a clear filter condition.|
|**Infrastructure complexity**|Adds an embedding model dependency (calling an external API or hosting one), a vector index, and often a separate query path alongside your normal relational queries.|
|**Dimensionality vs. performance**|Higher-dimensional embeddings usually capture more nuance but cost more to store, index, and search — there's a real trade-off between embedding quality and system performance.|

**49. What might go wrong if there is an index over the vector column?**
- **Approximate results**: most vector indexes (see below) are **approximate nearest neighbor (ANN)** structures, not exact — so an indexed similarity search may occasionally miss the true nearest match in exchange for speed. If you need guaranteed exact top-k results, you'd have to do a full sequential scan instead, which defeats the point of indexing at large scale.
- **Slow/expensive index builds**: building a vector index (e.g., HNSW) over a large table can be memory- and time-intensive, and needs tuning (parameters like `m`, `ef_construction` for HNSW) — poor tuning can lead to a slow build or a poor-quality index (bad accuracy/speed trade-off).
- **Index staleness / maintenance overhead**: every insert/update to a vector column may require the index to be updated, which adds write overhead; for IVFFlat-style indexes, the underlying data distribution ("clusters") can drift as new data comes in, degrading index quality over time until it's rebuilt.
- **Memory pressure**: some ANN index types (HNSW especially) hold significant structure in memory for fast lookups — on a large dataset, this can strain available RAM if not sized properly.
- **Dimension mismatch**: if vectors of inconsistent dimensionality are inserted (e.g., if you switch embedding models with different output sizes), you can get errors or corrupt/inconsistent index behavior.
- **Choosing the wrong distance metric for the index vs. your queries**: e.g., building an index optimized for L2 distance but querying with cosine distance means the index may not actually be used efficiently by the query planner, silently falling back to a slow scan.

**50. What type of index is used internally for vector search?**
The two main approximate nearest-neighbor (ANN) index types supported by pgvector:
- **IVFFlat (Inverted File with Flat compression)**
    
    - Splits the vector space into a number of clusters (using k-means-like partitioning).
    - At query time, it only searches the nearest cluster(s), which speeds things up compared to a full scan.
    - Needs to be trained/built after data exists (with a chosen number of lists), and needs to be told how many clusters to probe at query time (`probes`) — more probes = more accuracy, less speed.
    - Simpler and lighter on memory than HNSW, but generally less accurate/faster at query time for the same recall.
- **HNSW (Hierarchical Navigable Small World graphs)**
    
    - Builds a multi-layer graph structure connecting each vector to its nearest neighbors, allowing fast graph traversal to approximate the nearest neighbors of a query vector.
    - Generally offers better query performance and accuracy than IVFFlat, especially as data grows, but is more expensive to build and consumes more memory.
    - This is the more commonly recommended default for production pgvector use today.

Both are **approximate** — trading a small amount of accuracy for large speed gains compared to an exact brute-force scan (which compares the query vector against every row).

**51. What is the representation of the data stored with the vector type?**
In pgvector, a `vector(n)` column stores a **fixed-length array of floating-point numbers** (by default single-precision floats), where `n` is the declared dimensionality (e.g., `vector(1536)` for an OpenAI embedding model's output size). Internally:

- It's stored as a compact binary array of floats (not as JSON, not as text) — similar in spirit to PostgreSQL's native array types, but with a specialized type/format specific to pgvector for efficient distance computation.
- All vectors in a given column must have the **same fixed dimensionality**, since the column definition itself (`vector(n)`) hardcodes it.
- On the Java/Hibernate side, this maps naturally to a `float[]` (as shown in the `hibernate-vector` example above) — Hibernate/JDBC handles converting between the Java float array and pgvector's binary wire format.
- pgvector also supports variants like `halfvec` (half-precision floats, for reduced storage at some precision cost) and binary/bit vectors for specialized use cases (e.g., binary embeddings), in addition to the standard full-precision `vector` type.
