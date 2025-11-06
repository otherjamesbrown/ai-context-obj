# Spec 006: API Router Service

## Purpose

The main inference API that routes OpenAI-compatible requests to vLLM backends, enforces budgets, and logs usage asynchronously.

## Dependencies

- **003-database-schemas**: Tables for api_keys, model_mappings, usage_events
- **004-shared-libraries**: Auth, logging, metrics
- **005-user-org-service**: For API key validation
- **010-vllm-deployment**: vLLM endpoints available

## Success Criteria

- [ ] /v1/chat/completions endpoint 100% OpenAI-compatible
- [ ] API key authentication and validation working
- [ ] Budget checking before processing requests
- [ ] Request routing to correct vLLM backend
- [ ] Streaming responses (SSE format)
- [ ] Async usage logging to RabbitMQ
- [ ] P50 latency < 100ms (time to first token)
- [ ] Unit and integration tests

## User Stories

**As a Developer (Org User):**
- I want to send OpenAI-compatible requests using my API key
- I want to specify which model to use (llama3, mistral-7b)
- I want streaming responses for real-time output
- I want meaningful error messages when something fails

**As an Org Admin:**
- I want budget enforcement so runaway usage is prevented
- I want accurate usage tracking for cost analysis

## API Endpoints

### Inference API (OpenAI Compatible)
```
POST   /v1/chat/completions      # Main inference endpoint
GET    /v1/models                # List available models
GET    /v1/models/:id            # Get model details
```

### Internal Endpoints
```
GET    /health                   # Liveness probe
GET    /ready                    # Readiness probe
GET    /metrics                  # Prometheus metrics
```

## Request Flow

```
1. Client sends POST /v1/chat/completions with Bearer token (API key)
2. API Router validates API key
   - Hash key with SHA-256
   - Query api_keys table
   - Check key not revoked
3. Extract org_id from key
4. Check org budget (from Redis cache)
   - If exceeded → return 429 Too Many Requests
5. Lookup model backend URL from model_mappings (cached in Redis)
6. Forward request to vLLM backend
7. Stream response back to client
8. Count input/output tokens from response
9. Publish usage event to RabbitMQ (async)
10. Return final response to client
```

## Service Architecture

```
services/api-router/
├── cmd/
│   └── server/
│       └── main.go
│
├── internal/
│   ├── api/
│   │   ├── inference_handler.go    # Main handler
│   │   ├── models_handler.go       # List models
│   │   └── health_handler.go
│   │
│   ├── service/
│   │   ├── auth_service.go         # API key validation
│   │   ├── budget_service.go       # Budget checking
│   │   ├── router_service.go       # Route to vLLM
│   │   └── usage_service.go        # Usage logging
│   │
│   ├── repository/
│   │   ├── apikey_repository.go    # Query api_keys
│   │   ├── model_repository.go     # Query model_mappings
│   │   └── cache_repository.go     # Redis operations
│   │
│   ├── client/
│   │   └── vllm_client.go          # HTTP client for vLLM
│   │
│   └── queue/
│       └── publisher.go            # RabbitMQ publisher
│
├── tests/
│   ├── unit/
│   └── integration/
│
├── Dockerfile
├── Makefile
└── go.mod
```

## Key Implementation Details

### API Key Validation
```go
func (s *AuthService) ValidateAPIKey(ctx context.Context, key string) (*models.APIKeyInfo, error) {
    // Hash the provided key
    keyHash := sha256.Sum256([]byte(key))
    keyHashStr := hex.EncodeToString(keyHash[:])
    
    // Check cache first (Redis)
    if cached, ok := s.cache.Get("apikey:" + keyHashStr); ok {
        return cached.(*models.APIKeyInfo), nil
    }
    
    // Query database
    info, err := s.repo.GetByHash(ctx, keyHashStr)
    if err != nil {
        return nil, errors.ErrUnauthorized
    }
    
    // Check not revoked
    if info.RevokedAt != nil {
        return nil, errors.ErrUnauthorized
    }
    
    // Cache for 5 minutes
    s.cache.Set("apikey:"+keyHashStr, info, 5*time.Minute)
    
    // Update last_used_at asynchronously (don't block request)
    go s.repo.UpdateLastUsed(context.Background(), info.ID)
    
    return info, nil
}
```

### Budget Checking
```go
func (s *BudgetService) CheckBudget(ctx context.Context, orgID string, estimatedTokens int) error {
    // Get budget from Redis (cached from PostgreSQL)
    cacheKey := "budget:" + orgID
    budget, ok := s.cache.Get(cacheKey)
    
    if !ok {
        // Cache miss - query database
        org, err := s.repo.GetOrganization(ctx, orgID)
        if err != nil {
            return err
        }
        
        budget = org.BudgetTokensMonthly - org.TokensUsedCurrentMonth
        s.cache.Set(cacheKey, budget, 1*time.Minute) // Short TTL for budget
    }
    
    if budget.(int64) < int64(estimatedTokens) {
        return errors.ErrBudgetExceeded
    }
    
    return nil
}
```

### Model Routing
```go
func (s *RouterService) RouteRequest(ctx context.Context, modelName string, req *OpenAIRequest) (*http.Response, error) {
    // Get model backend URL from cache/database
    model, err := s.modelRepo.GetByName(ctx, modelName)
    if err != nil {
        return nil, fmt.Errorf("model not found: %s", modelName)
    }
    
    if !model.Enabled {
        return nil, fmt.Errorf("model disabled: %s", modelName)
    }
    
    // Forward request to vLLM backend
    backendURL := model.BackendURL + "/v1/chat/completions"
    return s.vllmClient.Forward(ctx, backendURL, req)
}
```

### Streaming Response
```go
func (h *InferenceHandler) handleChatCompletions(w http.ResponseWriter, r *http.Request) {
    // ... authentication, budget check, routing ...
    
    // Get streaming response from vLLM
    resp, err := h.router.RouteRequest(r.Context(), req.Model, req)
    if err != nil {
        errors.WriteProblemDetails(w, errors.ErrInternalServer)
        return
    }
    defer resp.Body.Close()
    
    // Set headers for SSE streaming
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")
    
    // Stream response chunks to client
    var inputTokens, outputTokens int
    scanner := bufio.NewScanner(resp.Body)
    
    for scanner.Scan() {
        line := scanner.Text()
        
        // Parse SSE data
        if strings.HasPrefix(line, "data: ") {
            data := strings.TrimPrefix(line, "data: ")
            
            if data == "[DONE]" {
                fmt.Fprintf(w, "data: [DONE]\n\n")
                w.(http.Flusher).Flush()
                break
            }
            
            // Parse chunk to count tokens
            var chunk OpenAIChunk
            json.Unmarshal([]byte(data), &chunk)
            
            // Forward to client
            fmt.Fprintf(w, "data: %s\n\n", data)
            w.(http.Flusher).Flush()
            
            // Track tokens
            if chunk.Usage != nil {
                inputTokens = chunk.Usage.PromptTokens
                outputTokens = chunk.Usage.CompletionTokens
            }
        }
    }
    
    // After streaming completes, publish usage event asynchronously
    go h.usageService.LogUsage(context.Background(), UsageEvent{
        OrgID:        keyInfo.OrgID,
        UserID:       keyInfo.UserID,
        APIKeyID:     keyInfo.ID,
        Model:        req.Model,
        InputTokens:  inputTokens,
        OutputTokens: outputTokens,
    })
}
```

### Usage Logging (Async)
```go
func (s *UsageService) LogUsage(ctx context.Context, event UsageEvent) error {
    // Calculate cost
    model, _ := s.modelRepo.GetByName(ctx, event.Model)
    cost := (float64(event.InputTokens) * model.CostPer1KInputTokens +
             float64(event.OutputTokens) * model.CostPer1KOutputTokens) / 1000.0
    
    // Create message
    msg := UsageMessage{
        EventID:      uuid.New(),
        Time:         time.Now(),
        OrgID:        event.OrgID,
        UserID:       event.UserID,
        APIKeyID:     event.APIKeyID,
        Model:        event.Model,
        InputTokens:  event.InputTokens,
        OutputTokens: event.OutputTokens,
        TotalTokens:  event.InputTokens + event.OutputTokens,
        Cost:         cost,
        Status:       "success",
    }
    
    // Publish to RabbitMQ
    return s.queue.Publish("usage-events", msg)
}
```

## OpenAI Compatibility

### Request Format
```json
POST /v1/chat/completions
Authorization: Bearer ias_xxxxxxxxxxxxx

{
  "model": "llama3",
  "messages": [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "Hello!"}
  ],
  "temperature": 0.7,
  "max_tokens": 2000,
  "stream": true
}
```

### Response Format (Streaming)
```
data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1234567890,"model":"llama3","choices":[{"index":0,"delta":{"content":"Hello"},"finish_reason":null}]}

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1234567890,"model":"llama3","choices":[{"index":0,"delta":{"content":"!"},"finish_reason":null}]}

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1234567890,"model":"llama3","choices":[{"index":0,"delta":{},"finish_reason":"stop"}],"usage":{"prompt_tokens":20,"completion_tokens":5,"total_tokens":25}}

data: [DONE]
```

### Error Response (RFC 7807)
```json
{
  "type": "https://platform.openai.com/docs/guides/error-codes/api-errors",
  "title": "Budget Exceeded",
  "status": 429,
  "detail": "Organization token budget exceeded for this month",
  "instance": "/v1/chat/completions"
}
```

## Configuration

```bash
PORT=8080
DATABASE_URL=postgres://...
REDIS_URL=redis://localhost:6379
RABBITMQ_URL=amqp://...
CACHE_TTL_MODELS=1h
CACHE_TTL_BUDGETS=1m
LOG_LEVEL=info
```

## Contracts

See `contracts/` for:
- `api-spec.yaml` - OpenAPI 3.0 specification
- `message-schemas.json` - RabbitMQ message formats
- `vllm-integration.md` - How to integrate with vLLM

## Testing Strategy

### Unit Tests
- API key validation logic
- Budget checking logic
- Token counting
- Error handling

### Integration Tests
- Full request flow with mock vLLM
- Budget enforcement
- Streaming responses
- Usage logging to queue

### Load Tests
- 1,000 concurrent requests
- P95 latency < 300ms
- No memory leaks

## Performance Optimization

### Caching Strategy
- **API keys**: Cache for 5 minutes (balance between performance and revocation delay)
- **Model mappings**: Cache for 1 hour (rarely change)
- **Org budgets**: Cache for 1 minute (need relatively fresh data)

### Connection Pooling
- Database: 50 max connections
- Redis: 10 connections
- RabbitMQ: 5 connections
- vLLM: HTTP client with connection pooling (100 max idle)

## Monitoring

### Key Metrics
- `api_router_requests_total{model, status}`
- `api_router_request_duration_seconds{model}` (histogram)
- `api_router_tokens_total{model, type="input|output"}`
- `api_router_budget_exceeded_total{org_id}`
- `api_router_vllm_errors_total{model}`

### Alerts
- Error rate > 5%
- P95 latency > 1 second
- RabbitMQ queue depth > 10,000

## Security

- API keys validated on every request
- No caching of revoked keys
- Budget checks prevent runaway costs
- Rate limiting per org (V2 feature)

## Open Questions

### Resolved
- ✅ Where to cache budgets? → Redis with 1-minute TTL
- ✅ How to handle vLLM failures? → Return 503, log error

### To Be Resolved
- Retry logic for vLLM failures? (Circuit breaker pattern in V2)
- Request queuing if vLLM busy? (Defer to V2)

## Next Steps

After this service is complete:
1. Implement **007-analytics-service** (consumes usage events from queue)
2. Implement **010-vllm-deployment** (deploy actual models)
3. Run E2E tests (spec 012)

## References

- OpenAI API Reference: https://platform.openai.com/docs/api-reference
- Server-Sent Events (SSE): https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events
- vLLM Documentation: https://docs.vllm.ai/

