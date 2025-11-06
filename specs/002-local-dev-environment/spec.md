# Spec 002: Local Development Environment

## Purpose

Provide a docker-compose setup for local development that mimics production without requiring Kubernetes or GPUs.

## Dependencies

- **000-project-setup**: Repository structure exists

## Success Criteria

- [ ] `docker-compose up` starts all dependencies
- [ ] PostgreSQL, Redis, RabbitMQ, mock inference service running
- [ ] Services can connect to all dependencies
- [ ] Database migrations run automatically on startup
- [ ] Developers can work offline (no cloud dependencies)

## Technical Scope

### Included
- PostgreSQL container (with TimescaleDB)
- Redis container
- RabbitMQ container with management UI
- Mock inference service (returns fake completions)
- Database initialization and seeding
- docker-compose configuration

### Excluded
- Actual vLLM deployment (use mock for local dev)
- Production services (those are deployed to K8s)
- Observability stack (LGTM - too heavy for local)

## Docker Compose Services

### Service: PostgreSQL
```yaml
postgres:
  image: timescale/timescaledb:latest-pg15
  environment:
    POSTGRES_DB: ias_dev
    POSTGRES_USER: ias_dev
    POSTGRES_PASSWORD: dev_password
  ports:
    - "5432:5432"
  volumes:
    - postgres_data:/var/lib/postgresql/data
    - ./migrations:/docker-entrypoint-initdb.d
```

### Service: Redis
```yaml
redis:
  image: redis:7-alpine
  ports:
    - "6379:6379"
  command: redis-server --appendonly yes
  volumes:
    - redis_data:/data
```

### Service: RabbitMQ
```yaml
rabbitmq:
  image: rabbitmq:3.12-management-alpine
  ports:
    - "5672:5672"   # AMQP
    - "15672:15672" # Management UI
  environment:
    RABBITMQ_DEFAULT_USER: ias_dev
    RABBITMQ_DEFAULT_PASS: dev_password
  volumes:
    - rabbitmq_data:/var/lib/rabbitmq
```

### Service: Mock Inference
Simple HTTP server that returns OpenAI-compatible fake responses.

```yaml
mock-inference:
  build: ./tests/mock-services/inference
  ports:
    - "8000:8000"
  environment:
    MODEL_NAME: mock-gpt
```

## Mock Inference Service Implementation

Simple Go HTTP server:

```go
// tests/mock-services/inference/main.go
package main

import (
    "encoding/json"
    "net/http"
    "time"
)

type ChatRequest struct {
    Model    string    `json:"model"`
    Messages []Message `json:"messages"`
    Stream   bool      `json:"stream"`
}

type Message struct {
    Role    string `json:"role"`
    Content string `json:"content"`
}

func handleChatCompletions(w http.ResponseWriter, r *http.Request) {
    var req ChatRequest
    json.NewDecoder(r.Body).Decode(&req)
    
    // Return mock response
    response := map[string]interface{}{
        "id": "mock-123",
        "object": "chat.completion",
        "created": time.Now().Unix(),
        "model": req.Model,
        "choices": []map[string]interface{}{
            {
                "index": 0,
                "message": map[string]string{
                    "role": "assistant",
                    "content": "This is a mock response from the development inference engine.",
                },
                "finish_reason": "stop",
            },
        },
        "usage": map[string]int{
            "prompt_tokens": 50,
            "completion_tokens": 15,
            "total_tokens": 65,
        },
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}

func main() {
    http.HandleFunc("/v1/chat/completions", handleChatCompletions)
    http.ListenAndServe(":8000", nil)
}
```

## Environment Configuration

### .env.local (for services)
```bash
DATABASE_URL=postgres://ias_dev:dev_password@localhost:5432/ias_dev?sslmode=disable
REDIS_URL=redis://localhost:6379
RABBITMQ_URL=amqp://ias_dev:dev_password@localhost:5672/
MOCK_INFERENCE_URL=http://localhost:8000
```

## Makefile Integration

```makefile
# Root Makefile additions
.PHONY: dev-up dev-down dev-logs dev-reset

dev-up:
	docker-compose up -d
	@echo "Waiting for services to be ready..."
	@sleep 5
	@echo "Running migrations..."
	migrate -path ./migrations -database "$(DATABASE_URL)" up
	@echo "✅ Development environment ready!"
	@echo "RabbitMQ UI: http://localhost:15672 (user: ias_dev, pass: dev_password)"

dev-down:
	docker-compose down

dev-logs:
	docker-compose logs -f

dev-reset:
	docker-compose down -v
	docker-compose up -d
```

## Database Seeding

On first startup, automatically seed with:
- Bootstrap org
- Test system admin user
- Mock model mapping

## Contracts

See `contracts/` for:
- `docker-compose.yml` - Complete compose file
- `mock-inference/` - Mock service implementation
- `.env.local.example` - Environment template
- `README.md` - Local development guide

## Testing

### Validation
- All containers start successfully
- Database migrations complete
- Services can connect to all dependencies
- Mock inference returns valid responses

## Next Steps

After local environment is set up:
1. Developers can implement services locally
2. Test against real PostgreSQL/Redis/RabbitMQ
3. No cloud dependencies for rapid iteration

