# LLM Gateway — Complete Project Reference

---

## Table of Contents
1. [What This Project Is](#1-what-this-project-is)
2. [Tech Stack](#2-tech-stack)
3. [Project Structure](#3-project-structure)
4. [The Two Big Parts](#4-the-two-big-parts)
5. [Database Schema](#5-database-schema)
6. [Entities & Relationships](#6-entities--relationships)
7. [Enums](#7-enums)
8. [Repository Layer](#8-repository-layer)
9. [Service Layer — All Methods](#9-service-layer--all-methods)
10. [Controller Layer — All Endpoints](#10-controller-layer--all-endpoints)
11. [DTOs](#11-dtos)
12. [The Caching Layer — How It Works](#12-the-caching-layer--how-it-works)
13. [Semantic Similarity — The Math](#13-semantic-similarity--the-math)
14. [Rate Limiting — Token Bucket Algorithm](#14-rate-limiting--token-bucket-algorithm)
15. [External API Integration](#15-external-api-integration)
16. [Full Request Flow — End to End](#16-full-request-flow--end-to-end)
17. [Validation Rules](#17-validation-rules)
18. [Exception Handling](#18-exception-handling)
19. [application.properties Configuration](#19-applicationproperties-configuration)
20. [Week 1 vs Week 2 Scope](#20-week-1-vs-week-2-scope)
21. [Common Mistakes to Avoid](#21-common-mistakes-to-avoid)
22. [Why Not a More Sophisticated Approach](#22-why-not-a-more-sophisticated-approach)

---

## 1. What This Project Is

An **LLM Gateway Backend** built with Spring Boot. It sits between client applications and one or more LLM providers (OpenAI, Ollama, Anthropic, etc.), and transparently makes every call cheaper, faster, and more reliable — without the client needing to know or care.

Think of it as a simplified internal version of LiteLLM, Portkey, or Helicone — the kind of service every team building on top of LLMs ends up needing once they have more than one client hitting the API.

**Core Goals:**
- Provide a single, OpenAI-compatible endpoint that any existing client SDK can point at with just a URL change
- Cache repeated (and semantically similar) prompts so the same question isn't paid for twice
- Enforce per-client rate limits before money is spent on an actual LLM call
- Track token usage and cost per API key
- Automatically fail over to a backup model/provider if the primary is down or rate-limited

---

## 2. Tech Stack

| Layer | Technology |
|-------|------------|
| Language | Java 17+ |
| Framework | Spring Boot |
| Database | MySQL (or H2 for local dev) |
| ORM | Spring Data JPA / Hibernate |
| HTTP Client | RestTemplate or WebClient |
| External APIs | OpenAI API, Ollama local REST API, embedding provider (OpenAI `text-embedding-3-small` or a local embedding model) |
| Build Tool | Maven or Gradle |
| Utilities | Lombok |

---

## 3. Project Structure

```
src/main/java/com/gateway/
│
├── entity/
│   ├── ApiKeyEntity.java           ✅ done
│   ├── CacheEntryEntity.java
│   ├── UsageLogEntity.java
│   ├── BackendConfigEntity.java
│   └── enums/
│       ├── ApiKeyStatus.java
│       ├── BackendStatus.java
│       └── RequestOutcome.java
│
├── repository/
│   ├── ApiKeyRepository.java       ✅ done
│   ├── CacheEntryRepository.java
│   ├── UsageLogRepository.java
│   └── BackendConfigRepository.java
│
├── service/
│   ├── ApiKeyService.java
│   ├── ExactCacheService.java
│   ├── SemanticCacheService.java
│   ├── EmbeddingService.java
│   ├── RateLimiterService.java
│   ├── BackendRouterService.java
│   ├── CostTrackerService.java
│   └── GatewayService.java
│
├── controller/
│   ├── ChatCompletionController.java
│   ├── ApiKeyController.java
│   └── UsageController.java
│
├── dto/
│   ├── ChatCompletionRequestDTO.java
│   ├── ChatCompletionResponseDTO.java
│   ├── ApiKeyRequestDTO.java
│   ├── ApiKeyResponseDTO.java
│   ├── UsageResponseDTO.java
│   └── CacheHitMetadataDTO.java
│
├── config/
│   ├── BackendConfig.java
│   └── RestTemplateConfig.java
│
└── exception/
    ├── GlobalExceptionHandler.java
    ├── RateLimitExceededException.java
    ├── BackendUnavailableException.java
    └── InvalidApiKeyException.java
```

---

## 4. The Two Big Parts

### Part 1 — Client & Usage Registry (CRUD / Data Layer)
Structured database management for:
- **API Keys** — client identity, budget limits, status (active/suspended)
- **Backend Configs** — which providers are configured, their priority order, credentials
- **Usage Logs** — one row per request: tokens in/out, cost, cache hit or miss, which backend served it
- **Cache Entries** — stored prompt/response pairs, with embeddings for semantic lookup

No smart logic here. Just keeping records clean, valid, and queryable.

### Part 2 — Gateway Engine (Smart Layer)
Given an incoming chat completion request:
1. Authenticate the API key and check its budget/status
2. Check the rate limiter — reject early if the client is over their limit
3. Check the cache — exact match first, then semantic match
4. On a cache miss, route to the best available backend (with fallback)
5. Record token usage and cost against that client
6. Return the response, and update the cache for next time

---

## 5. Database Schema

```sql
-- API Keys
CREATE TABLE api_key (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    key_value VARCHAR(255) UNIQUE NOT NULL,
    client_name VARCHAR(255) NOT NULL,
    monthly_budget_usd DOUBLE,
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Backend Configs
CREATE TABLE backend_config (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    base_url VARCHAR(500) NOT NULL,
    priority INT NOT NULL,
    status VARCHAR(50),
    input_price_per_1k DOUBLE,
    output_price_per_1k DOUBLE
);

-- Cache Entries
CREATE TABLE cache_entry (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    prompt_hash VARCHAR(255) NOT NULL,
    prompt_text TEXT NOT NULL,
    response_text TEXT NOT NULL,
    embedding TEXT,
    model VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP
);

-- Usage Logs
CREATE TABLE usage_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    api_key_id BIGINT NOT NULL,
    backend_name VARCHAR(100),
    input_tokens INT,
    output_tokens INT,
    cost_usd DOUBLE,
    cache_hit BOOLEAN DEFAULT FALSE,
    outcome VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (api_key_id) REFERENCES api_key(id)
);
```

> Note: With `spring.jpa.hibernate.ddl-auto=update`, Hibernate auto-generates these tables. You don't need to run this SQL manually — it's here for understanding the schema.

---

## 6. Entities & Relationships

- **ApiKeyEntity** — one-to-many with `UsageLogEntity` (a key generates many usage log rows over time).
- **BackendConfigEntity** — standalone; the router reads all rows ordered by `priority` to decide call order.
- **CacheEntryEntity** — standalone; looked up by `prompt_hash` for exact match, and by embedding similarity for semantic match. `expires_at` drives TTL eviction.
- **UsageLogEntity** — many-to-one with `ApiKeyEntity`. This table is what the usage/cost dashboard aggregates over.

Store the `embedding` column as a serialized float array (e.g. comma-separated string or JSON) since plain MySQL has no native vector type at this scope — a dedicated vector column type is a reasonable future extension, not a Week 1/2 requirement.

---

## 7. Enums

```java
public enum ApiKeyStatus {
    ACTIVE, SUSPENDED, OVER_BUDGET
}

public enum BackendStatus {
    HEALTHY, DEGRADED, DOWN
}

public enum RequestOutcome {
    SUCCESS, CACHE_HIT, RATE_LIMITED, BACKEND_FAILURE, ALL_BACKENDS_FAILED
}
```

Always use `@Enumerated(EnumType.STRING)` — see [Common Mistakes](#21-common-mistakes-to-avoid).

---

## 8. Repository Layer

```java
public interface ApiKeyRepository extends JpaRepository<ApiKeyEntity, Long> {
    Optional<ApiKeyEntity> findByKeyValue(String keyValue);
}

public interface CacheEntryRepository extends JpaRepository<CacheEntryEntity, Long> {
    Optional<CacheEntryEntity> findByPromptHashAndModel(String promptHash, String model);
    List<CacheEntryEntity> findByModelAndExpiresAtAfter(String model, LocalDateTime now);
}

public interface UsageLogRepository extends JpaRepository<UsageLogEntity, Long> {
    List<UsageLogEntity> findByApiKeyIdAndCreatedAtBetween(Long apiKeyId, LocalDateTime start, LocalDateTime end);

    @Query("SELECT SUM(u.costUsd) FROM UsageLogEntity u WHERE u.apiKeyId = :apiKeyId AND u.createdAt >= :since")
    Double sumCostSince(Long apiKeyId, LocalDateTime since);
}

public interface BackendConfigRepository extends JpaRepository<BackendConfigEntity, Long> {
    List<BackendConfigEntity> findByStatusOrderByPriorityAsc(BackendStatus status);
}
```

---

## 9. Service Layer — All Methods

**ApiKeyService**
- `validateAndFetch(String keyValue)` — throws `InvalidApiKeyException` if not found or suspended
- `checkBudget(ApiKeyEntity key)` — throws if monthly spend exceeds `monthlyBudgetUsd`

**ExactCacheService**
- `lookup(String promptHash, String model)` — returns `Optional<CacheEntryEntity>`
- `store(String prompt, String response, String model)` — hashes and persists
- `evictExpired()` — scheduled cleanup, runs on a timer

**SemanticCacheService**
- `findSimilar(float[] promptEmbedding, String model, double threshold)` — scans cached embeddings for the given model, returns best match above threshold or `Optional.empty()`
- `store(String prompt, float[] embedding, String response, String model)`

**EmbeddingService**
- `embed(String text)` — calls the embedding provider, returns `float[]`

**RateLimiterService**
- `allow(String apiKey)` — token bucket check, returns boolean; throws `RateLimitExceededException` if exhausted

**BackendRouterService**
- `getOrderedBackends()` — healthy backends sorted by priority
- `callWithFallback(ChatCompletionRequestDTO request)` — tries each backend in order until one succeeds or all fail

**CostTrackerService**
- `computeCost(String backendName, int inputTokens, int outputTokens)` — looks up pricing, returns cost
- `logUsage(Long apiKeyId, String backendName, int inputTokens, int outputTokens, double cost, boolean cacheHit, RequestOutcome outcome)`

**GatewayService** (the orchestrator, mirrors `DispatchService` in the fleet project)
- `handleChatCompletion(String apiKey, ChatCompletionRequestDTO request)` — runs the full flow described in [Section 16](#16-full-request-flow--end-to-end)

---

## 10. Controller Layer — All Endpoints

```java
POST   /v1/chat/completions        // main gateway endpoint, OpenAI-compatible body
GET    /v1/usage/{apiKey}          // usage/cost summary for a client
POST   /v1/admin/api-keys          // create a new API key
GET    /v1/admin/api-keys          // list all API keys
PATCH  /v1/admin/api-keys/{id}     // suspend / update budget
GET    /v1/admin/backends          // list configured backends and health status
```

---

## 11. DTOs

```java
public record ChatCompletionRequestDTO(String model, List<Message> messages) {
    public record Message(String role, String content) {}
}

public record ChatCompletionResponseDTO(
    String id,
    String model,
    List<Choice> choices,
    Usage usage,
    CacheHitMetadataDTO cacheMetadata
) {
    public record Choice(int index, Message message, String finishReason) {}
    public record Message(String role, String content) {}
    public record Usage(int promptTokens, int completionTokens, int totalTokens) {}
}

public record CacheHitMetadataDTO(boolean hit, String matchType, Double similarityScore) {}

public record ApiKeyRequestDTO(String clientName, Double monthlyBudgetUsd) {}

public record ApiKeyResponseDTO(Long id, String keyValue, String clientName, String status) {}

public record UsageResponseDTO(
    String apiKey,
    double totalCostUsd,
    int totalRequests,
    int cacheHits,
    double cacheHitRate
) {}
```

Using Java records for DTOs keeps them immutable and removes boilerplate getters/setters — a clean fit here since none of these need mutable state after construction.

---

## 12. The Caching Layer — How It Works

Two tiers, checked in order:

**Tier 1 — Exact match.** Normalize the incoming prompt (trim whitespace, collapse case if appropriate), compute a hash (e.g. SHA-256), and look up `CacheEntryRepository.findByPromptHashAndModel(...)`. If found and not expired, return immediately — zero cost, near-zero latency.

**Tier 2 — Semantic match.** If no exact match, embed the prompt via `EmbeddingService`, then compare against the embeddings of recently cached prompts for the same model using cosine similarity. If the best match exceeds a configured threshold (e.g. 0.92–0.95), treat it as a hit and return that cached response.

Both tiers write back to the cache on a miss: after the backend responds, store the prompt (hash + text + embedding) alongside the response, with an expiry timestamp (TTL).

**Why two tiers, not just one:** exact match is essentially free to compute (a hash lookup) and has zero false-positive risk, so it should always run first. Semantic match costs an embedding API call and carries real risk of returning a *wrong* answer if the threshold is too loose — it's a more powerful but more expensive and riskier tool, so it's only reached when the cheap, safe option misses.

---

## 13. Semantic Similarity — The Math

Cosine similarity between two embedding vectors **a** and **b**:

```
cosine_similarity(a, b) = (a · b) / (||a|| * ||b||)
```

Where `a · b` is the dot product and `||a||` is the Euclidean norm (magnitude) of vector `a`.

```java
public static double cosineSimilarity(float[] a, float[] b) {
    double dot = 0.0, normA = 0.0, normB = 0.0;
    for (int i = 0; i < a.length; i++) {
        dot += a[i] * b[i];
        normA += a[i] * a[i];
        normB += b[i] * b[i];
    }
    return dot / (Math.sqrt(normA) * Math.sqrt(normB));
}
```

Result ranges from -1 to 1; in practice, for text embeddings of genuinely related prompts, values land between roughly 0.7 (loosely related) and 0.99 (near-identical meaning). The threshold you pick here is the single most important tuning knob in the whole project — too low and the cache starts returning wrong answers for different questions; too high and the semantic tier rarely fires at all, providing no benefit over exact match alone. This tuning is a legitimate experiment worth its own section in a report: run a labeled set of prompt pairs (some truly equivalent, some superficially similar but different), sweep the threshold, and plot precision (correct cache hits) against recall (how many equivalent prompts were actually caught).

**Brute-force comparison** (compare the incoming embedding against every cached entry for that model) is the correct starting point given the expected scale — cache sizes in the tens or low hundreds of entries make this trivially fast. If cache size grows into the thousands, an approximate nearest-neighbor index (e.g. a simple LSH bucket scheme) is the natural next step, but it is not needed to have a correct, working, demoable system at this project's scale.

---

## 14. Rate Limiting — Token Bucket Algorithm

Each API key gets a bucket with a maximum capacity and a refill rate.

```
function allow(apiKey):
    bucket = getBucket(apiKey)
    now = currentTime()
    elapsed = now - bucket.lastRefill
    bucket.tokens = min(bucket.capacity, bucket.tokens + elapsed * bucket.refillRate)
    bucket.lastRefill = now

    if bucket.tokens >= 1:
        bucket.tokens -= 1
        return true
    else:
        return false
```

**Complexity:** O(1) per check.
**Role in this project:** runs before any cache lookup or backend call — rejecting an over-limit request at this stage means zero wasted embedding calls and zero wasted LLM spend on a request that was going to be rejected anyway.

Store buckets in a `ConcurrentHashMap<String, TokenBucket>` keyed by API key. Since Spring's default request-handling model already gives each request its own thread, and the only shared state here is per-key bucket counters, a `ConcurrentHashMap` plus `synchronized` (or `AtomicLong`-based counters) is sufficient — no need for anything more elaborate at this scale.

---

## 15. External API Integration

**OpenAI-compatible chat completion** — the gateway's own `/v1/chat/completions` endpoint mirrors OpenAI's request/response shape, so any existing OpenAI SDK can point at the gateway by changing only its base URL.

**Backend calls** — `BackendRouterService` calls whichever backend is next in priority order using `RestTemplate` or `WebClient`, translating the gateway's internal request format into whatever shape that specific backend expects (OpenAI's API, Ollama's local REST API, etc. — each backend may need a small adapter).

**Embedding calls** — a separate external call (or a local embedding model via Ollama) used only by `EmbeddingService`, decoupled from the main completion call so a slow or failing embedding provider degrades semantic caching gracefully rather than breaking the whole gateway (fall back to exact-match-only caching if embeddings are unavailable).

**Fallback logic** — if a call to the primary backend times out or returns a 429/5xx, immediately mark it degraded for a short cooldown window and retry against the next backend in priority order, rather than waiting for a slow timeout to surface all the way back to the client.

---

## 16. Full Request Flow — End to End

1. **Client sends `POST /v1/chat/completions`** with an API key in the header and an OpenAI-shaped JSON body.
2. **`ApiKeyService.validateAndFetch`** — confirm the key exists and is `ACTIVE`; check budget via `checkBudget`.
3. **`RateLimiterService.allow`** — reject with 429 immediately if the client's bucket is empty.
4. **`ExactCacheService.lookup`** — hash the normalized prompt, check for an exact cached match.
5. **On exact hit** → return cached response, log usage with `cacheHit=true`, `outcome=CACHE_HIT`, cost = 0. Done.
6. **On exact miss** → `EmbeddingService.embed` the prompt, then `SemanticCacheService.findSimilar`.
7. **On semantic hit** → return the matched cached response, log usage with cache-hit metadata including the similarity score. Done.
8. **On full cache miss** → `BackendRouterService.callWithFallback` tries backends in priority order.
9. **Backend responds** → parse token counts from its response, `CostTrackerService.computeCost` using that backend's pricing.
10. **Store the new prompt/response pair** in both cache tiers (hash + embedding) with a TTL.
11. **`CostTrackerService.logUsage`** — persist the usage row against this API key.
12. **Return the response to the client**, in the OpenAI-compatible shape.

---

## 17. Validation Rules

- API key must exist and be `ACTIVE` before any other processing happens (fail fast).
- Request body must include a non-empty `model` field and at least one message.
- `monthlyBudgetUsd` on key creation must be a positive number.
- Backend `priority` values must be unique per active backend (ties would make fallback order ambiguous).
- Semantic similarity threshold must be configurable per-model, not hardcoded — different embedding models produce different similarity distributions.

---

## 18. Exception Handling

### Global Handler
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(InvalidApiKeyException.class)
    public ResponseEntity<String> handleInvalidKey(InvalidApiKeyException ex) {
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body(ex.getMessage());
    }

    @ExceptionHandler(RateLimitExceededException.class)
    public ResponseEntity<String> handleRateLimit(RateLimitExceededException ex) {
        return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS).body(ex.getMessage());
    }

    @ExceptionHandler(BackendUnavailableException.class)
    public ResponseEntity<String> handleBackendDown(BackendUnavailableException ex) {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE).body(ex.getMessage());
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<String> handleGeneral(Exception ex) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("Something went wrong");
    }
}
```

### Custom Exceptions
```java
public class InvalidApiKeyException extends RuntimeException {
    public InvalidApiKeyException(String message) { super(message); }
}

public class RateLimitExceededException extends RuntimeException {
    public RateLimitExceededException(String message) { super(message); }
}

public class BackendUnavailableException extends RuntimeException {
    public BackendUnavailableException(String message) { super(message); }
}
```

---

## 19. application.properties Configuration

```properties
# Database
spring.datasource.url=jdbc:mysql://localhost:3306/gateway_db
spring.datasource.username=root
spring.datasource.password=your_password
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# JPA / Hibernate
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.database-platform=org.hibernate.dialect.MySQLDialect

# Backends
backend.openai.api-key=YOUR_OPENAI_API_KEY
backend.openai.base-url=https://api.openai.com/v1/chat/completions
backend.ollama.base-url=http://localhost:11434/api/chat

# Embeddings
embedding.provider.base-url=https://api.openai.com/v1/embeddings
embedding.provider.api-key=YOUR_OPENAI_API_KEY
embedding.similarity.threshold=0.93

# Rate limiting defaults
ratelimit.default.capacity=60
ratelimit.default.refill-per-second=1.0

# App
server.port=8080
```

---

## 20. Week 1 vs Week 2 Scope

### Week 1 — You Are Here
- [x] ApiKeyEntity + ApiKeyRepository
- [ ] CacheEntryEntity, UsageLogEntity, BackendConfigEntity
- [ ] All Enums
- [ ] CacheEntryRepository, UsageLogRepository, BackendConfigRepository
- [ ] ApiKeyService (basic CRUD + validation only)
- [ ] ApiKeyController, admin endpoints for managing keys/backends
- [ ] Basic validation rules
- [ ] Exception handling setup
- [ ] application.properties + MySQL connection

### Week 2 — Coming Next
- [ ] ExactCacheService (hash-based lookup + storage)
- [ ] RateLimiterService (token bucket)
- [ ] BackendRouterService (priority order + fallback)
- [ ] CostTrackerService (pricing table + usage logging)
- [ ] GatewayService (orchestrator)
- [ ] ChatCompletionController
- [ ] All DTOs
- [ ] RestTemplate config
- [ ] Full request flow end-to-end
- [ ] EmbeddingService + SemanticCacheService (semantic caching layer)
- [ ] Threshold tuning experiment + results

---

## 21. Common Mistakes to Avoid

| Mistake | Why It's a Problem | Fix |
|---------|--------------------|-----|
| Using `@Enumerated(EnumType.ORDINAL)` | Stores 0,1,2 in DB — breaks if you reorder enum values | Always use `EnumType.STRING` |
| Checking semantic cache before exact cache | Wastes an embedding API call when a free hash lookup would have hit | Always check exact match first |
| Hardcoding the similarity threshold | Different embedding models have different similarity distributions — one threshold doesn't fit all | Make it configurable per-model |
| Checking cache after calling the rate limiter incorrectly (or skipping rate limit entirely) | A client can burn through requests before hitting any cost-saving logic | Rate limit check must run before cache lookup and before any backend call |
| No fallback across backends | If the primary provider is down or rate-limited, every request fails | Always implement ordered fallback |
| Returning entities directly from controllers | Exposes DB structure, leaks internal fields | Always use DTOs in controllers |
| Not setting a TTL on cache entries | Stale answers get served indefinitely, including for prompts whose "correct" answer may have changed | Always set `expires_at` and run scheduled eviction |
| Computing cost before parsing actual token counts from the backend response | Estimated token counts can be significantly wrong | Always use the token counts the provider actually returns |
| Hardcoding API keys in code | Security risk | Always use `application.properties` + environment variables |
| Skipping `@Transactional` on usage-logging writes | Partial writes possible if an error occurs mid-log | Add `@Transactional` to write operations |

---

## 22. Why Not a More Sophisticated Approach

**On caching:** a proper vector database (Pinecone, Weaviate, Milvus) would be the production-grade choice for semantic caching at scale, using an approximate nearest-neighbor index like HNSW. At this project's expected cache size (low hundreds to low thousands of entries), brute-force cosine similarity comparison is fast enough that an ANN index would add real implementation complexity (index construction, tuning, memory layout) without a measurable latency benefit. Worth revisiting if cache size grows into the tens of thousands.

**On rate limiting:** a distributed rate limiter backed by Redis (so limits are enforced consistently across multiple gateway instances) is the right choice once this service needs to scale horizontally. For a single-instance deployment, an in-memory token bucket per API key is correct, simpler, and has zero external dependency — adding Redis here would be solving a scaling problem this project doesn't yet have.

**On routing:** a more sophisticated router could account for live latency/cost trade-offs per backend (similar in spirit to a learned router) and pick dynamically rather than by static priority order. Static priority with health-based fallback is deliberately simpler and fully sufficient for a small number of configured backends — a learned or adaptive router is a reasonable future extension if the number of backends and their relative performance becomes harder to reason about by hand.

## Future Improvements

- Move rate limiting to Redis if the gateway needs to run as more than one instance.
- Add an approximate nearest-neighbor index for semantic cache lookup if cache size grows significantly.
- Track cache-hit rate and cost savings over time on a small usage dashboard, broken down per API key.
- Consider a learned/adaptive backend router if latency or cost data suggests static priority ordering is leaving savings on the table.
