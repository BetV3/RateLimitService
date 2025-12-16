# Rate Limiter Service

A production-grade, distributed rate limiting service built in Go. Supports multiple algorithms, Redis-backed distributed limiting, and Prometheus metrics.

![Go Version](https://img.shields.io/badge/Go-1.21+-00ADD8?style=flat&logo=go)
![License](https://img.shields.io/badge/license-MIT-green)
![Build Status](https://img.shields.io/badge/build-passing-brightgreen)

## Why This Exists

Every API needs rate limiting. This service provides a centralized, horizontally-scalable rate limiter that can sit in front of any API gateway or be called directly from your services.

**Features:**
- **Multiple algorithms** — Fixed window, sliding window counter, and token bucket
- **Distributed** — Redis backend with Lua scripts for atomic operations
- **Multi-tenant** — Different rate limits per API key, user, or endpoint
- **Observable** — Prometheus metrics, structured logging, health checks
- **Fast** — Sub-millisecond latency for in-memory, < 5ms for Redis

## Quick Start

### Using Docker Compose

```bash
git clone https://github.com/BetV3/rate-limiter.git
cd rate-limiter
docker-compose up
```

The service will be available at `http://localhost:8080`.

### Local Development

```bash
# Start Redis
docker run -d -p 6379:6379 redis:7-alpine

# Run the service
go run cmd/server/main.go
```

## Architecture

```
┌──────────────┐     ┌──────────────────┐     ┌─────────────┐
│   Client     │────▶│  Rate Limiter    │────▶│   Redis     │
│  (API GW)    │◀────│  Service (Go)    │◀────│   Cluster   │
└──────────────┘     └──────────────────┘     └─────────────┘
                              │
                              ▼
                     ┌────────────────┐
                     │  Prometheus    │
                     └────────────────┘
```

The service is stateless — all rate limit state lives in Redis, allowing horizontal scaling. Lua scripts ensure atomic check-and-decrement operations to prevent race conditions.

## API Reference

### Check Rate Limit

```http
POST /api/v1/check
Content-Type: application/json

{
  "key": "user:12345",
  "limit": 100,
  "window": "1m"
}
```

**Response (200 OK - Allowed):**
```json
{
  "allowed": true,
  "remaining": 94,
  "reset_at": "2025-01-15T10:31:00Z",
  "limit": 100
}
```

**Response (429 Too Many Requests - Denied):**
```json
{
  "allowed": false,
  "remaining": 0,
  "reset_at": "2025-01-15T10:31:00Z",
  "limit": 100,
  "retry_after": 45
}
```

**Response Headers:**
| Header | Description |
|--------|-------------|
| `X-RateLimit-Limit` | Maximum requests allowed in window |
| `X-RateLimit-Remaining` | Requests remaining in current window |
| `X-RateLimit-Reset` | Unix timestamp when the window resets |
| `Retry-After` | Seconds until next request allowed (only on 429) |

### Get Rate Limit Status

```http
GET /api/v1/status/:key
```

Returns current rate limit status for a key without decrementing the counter.

### Update Configuration

```http
POST /api/v1/config
Content-Type: application/json

{
  "rules": [
    {
      "name": "default",
      "limit": 100,
      "window": "1m",
      "algorithm": "sliding_window"
    },
    {
      "name": "premium",
      "limit": 1000,
      "window": "1m",
      "algorithm": "token_bucket",
      "burst": 50
    }
  ]
}
```

### Health Check

```http
GET /health
```

Returns `200 OK` if the service is healthy, `503 Service Unavailable` if Redis is unreachable.

### Metrics

```http
GET /metrics
```

Prometheus-formatted metrics endpoint.

## Rate Limiting Algorithms

### Fixed Window

Divides time into fixed windows (e.g., every minute starting at :00). Simple and memory-efficient, but can allow up to 2x the rate limit at window boundaries.

```
Window 1 (10:00-10:01)    Window 2 (10:01-10:02)
[----100 requests----]    [----100 requests----]
                     ↑    ↑
              Boundary: 200 requests possible in 1 second
```

**Best for:** Simple use cases where exact precision isn't critical.

### Sliding Window Counter

Combines the current window count with a weighted portion of the previous window. Smooths out the boundary issue while remaining memory-efficient.

```
Rate = (prev_window_count * overlap_percentage) + current_window_count
```

**Best for:** Most production use cases. Good balance of accuracy and efficiency.

### Token Bucket

Tokens are added to a bucket at a fixed rate. Each request consumes a token. Allows controlled bursts up to the bucket size.

```
Bucket Size: 10 tokens
Refill Rate: 1 token/second

[##########]  → Full bucket, can burst 10 requests
[####------]  → 4 tokens remaining
[----------]  → Empty, must wait for refill
```

**Best for:** APIs where you want to allow short bursts while enforcing average rate.

## Configuration

Configuration via environment variables or config file:

```yaml
# config.yaml
server:
  port: 8080
  read_timeout: 5s
  write_timeout: 10s

redis:
  address: localhost:6379
  password: ""
  db: 0
  pool_size: 10

limiter:
  default_algorithm: sliding_window
  default_limit: 100
  default_window: 1m

logging:
  level: info
  format: json

metrics:
  enabled: true
  path: /metrics
```

**Environment Variables:**

| Variable | Description | Default |
|----------|-------------|---------|
| `PORT` | Server port | `8080` |
| `REDIS_URL` | Redis connection URL | `localhost:6379` |
| `LOG_LEVEL` | Log level (debug, info, warn, error) | `info` |
| `DEFAULT_LIMIT` | Default rate limit | `100` |
| `DEFAULT_WINDOW` | Default window duration | `1m` |

## Performance

Benchmarks run on M1 MacBook Pro, Redis 7.0:

| Scenario | Throughput | p50 Latency | p99 Latency |
|----------|------------|-------------|-------------|
| In-memory, fixed window | 150,000 req/s | 0.1ms | 0.3ms |
| Redis, fixed window | 45,000 req/s | 1.2ms | 3.1ms |
| Redis, sliding window | 38,000 req/s | 1.5ms | 4.2ms |
| Redis, token bucket | 42,000 req/s | 1.3ms | 3.5ms |

Run benchmarks locally:

```bash
make bench
```

## Development

### Prerequisites

- Go 1.21+
- Docker and Docker Compose
- Make

### Setup

```bash
# Clone the repo
git clone https://github.com/BetV3/rate-limiter.git
cd rate-limiter

# Install dependencies
go mod download

# Start dependencies
docker-compose up -d redis

# Run tests
make test

# Run with hot reload (requires air)
make dev
```

### Project Structure

```
rate-limiter/
├── cmd/
│   └── server/
│       └── main.go           # Application entrypoint
├── internal/
│   ├── limiter/              # Rate limiting algorithms
│   │   ├── interface.go
│   │   ├── fixedwindow.go
│   │   ├── slidingwindow.go
│   │   └── tokenbucket.go
│   ├── storage/              # Storage backends
│   │   ├── memory.go
│   │   ├── redis.go
│   │   └── lua/              # Redis Lua scripts
│   ├── api/                  # HTTP handlers and routing
│   ├── config/               # Configuration loading
│   └── metrics/              # Prometheus metrics
├── docker-compose.yml
├── Dockerfile
├── Makefile
└── README.md
```

### Running Tests

```bash
# Unit tests
make test

# Integration tests (requires Docker)
make test-integration

# Coverage report
make coverage
```

### Makefile Commands

| Command | Description |
|---------|-------------|
| `make build` | Build the binary |
| `make run` | Run the service |
| `make dev` | Run with hot reload |
| `make test` | Run unit tests |
| `make test-integration` | Run integration tests |
| `make bench` | Run benchmarks |
| `make lint` | Run linter |
| `make docker-build` | Build Docker image |

## Deployment

### Docker

```bash
docker build -t rate-limiter .
docker run -p 8080:8080 -e REDIS_URL=redis:6379 rate-limiter
```

### Kubernetes

Helm chart coming soon. For now, see `deploy/kubernetes/` for example manifests.

### Health Checks

- **Liveness:** `GET /health` — Returns 200 if the process is running
- **Readiness:** `GET /ready` — Returns 200 if Redis is connected and service can accept traffic

## Observability

### Prometheus Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `ratelimiter_requests_total` | Counter | Total requests checked |
| `ratelimiter_requests_allowed_total` | Counter | Requests that were allowed |
| `ratelimiter_requests_denied_total` | Counter | Requests that were denied |
| `ratelimiter_check_duration_seconds` | Histogram | Latency of rate limit checks |
| `ratelimiter_redis_errors_total` | Counter | Redis operation errors |

### Example Grafana Dashboard

Import `dashboards/rate-limiter.json` into Grafana for a pre-built dashboard.

### Structured Logging

All logs are JSON-formatted for easy parsing:

```json
{
  "level": "info",
  "time": "2025-01-15T10:30:45Z",
  "msg": "rate limit checked",
  "key": "user:12345",
  "allowed": true,
  "remaining": 94,
  "algorithm": "sliding_window",
  "latency_ms": 1.2
}
```

## Design Decisions

### Why Lua Scripts for Redis?

Redis operations need to be atomic to prevent race conditions. Consider two concurrent requests:

```
Request A: GET count → 99
Request B: GET count → 99
Request A: SET count → 100 ✓ (allowed)
Request B: SET count → 100 ✓ (allowed, but should be denied!)
```

Lua scripts execute atomically in Redis, preventing this issue:

```lua
local count = redis.call('INCR', KEYS[1])
if count == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[1])
end
if count > tonumber(ARGV[2]) then
    return {0, count, redis.call('TTL', KEYS[1])}
end
return {1, tonumber(ARGV[2]) - count, redis.call('TTL', KEYS[1])}
```

### Why Interface-Based Storage?

Allows easy swapping between in-memory (development/testing) and Redis (production) without changing business logic. Also enables future backends (Memcached, DynamoDB) without major refactoring.

### Why Not Middleware?

This is a standalone service rather than middleware because:
- Can be shared across multiple services/languages
- Scales independently of your application
- Single source of truth for rate limits
- Easier to update rate limiting logic without redeploying all services

## Roadmap

- [x] Fixed window algorithm
- [x] Sliding window counter
- [x] Token bucket
- [x] Redis backend
- [x] Prometheus metrics
- [ ] gRPC API
- [ ] Admin dashboard UI
- [ ] Sliding window log algorithm
- [ ] Cluster mode (consistent hashing)
- [ ] Rate limit by multiple dimensions

## Contributing

Contributions are welcome! Please read [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

MIT License. See [LICENSE](LICENSE) for details.

## Acknowledgments

- [Token bucket algorithm](https://en.wikipedia.org/wiki/Token_bucket)
- [Stripe's rate limiting blog post](https://stripe.com/blog/rate-limiters)
- [Cloudflare's approach to rate limiting](https://blog.cloudflare.com/counting-things-a-lot-of-different-things/)
