# Notirail Gateway

The Gateway Service is the single entry point for all external client traffic into Notirail. It handles authentication, tenant identification, rate limiting, request validation, and routing to internal services (e.g., Message Ingestion, Status API, Admin/Dashboard APIs). It is a stateless Spring Boot 4 application running on Java 25.

## Goals & Non-Goals

**Goals**
- Centralized security and policy enforcement (auth, quotas, CORS, etc.).  
- Clear separation between public API traffic and internal/admin traffic.  
- Low-latency request handling with robust error handling and observability.  
- Easy evolution of internal services without breaking external API contracts.  

**Non-Goals**
- Business logic for message delivery or provider integrations.  
- Persistent storage of domain data (beyond minimal caching / rate-limit state).  

## Responsibilities

- **Authentication & Authorization**  
  - Validate API keys / tokens for `POST /v1/messages`, `GET /v1/messages/{id}`, etc.  
  - Resolve `tenant_id` from credentials and attach to request context / headers.  
  - Enforce scope-based access (e.g., `messages:send`, `messages:read`, `admin:*`).  

- **Rate Limiting & Quotas**  
  - Enforce per-tenant and per-API-key rate limits (requests/sec, daily volume).  
  - Return `429 Too Many Requests` with retry-after information when limits are exceeded.  

- **Request Validation**  
  - Validate HTTP method, path, headers (e.g., `Content-Type`, `Idempotency-Key`).  
  - Validate JSON schema for request bodies (e.g., message submission payload).  
  - Normalize and sanitize inputs before forwarding to internal services.  

- **Routing & Protocol Translation**  
  - Route requests to appropriate internal services (Ingestion, Status, Admin).  
  - Optionally translate between external REST and internal gRPC or other protocols.  
  - Manage versioning (`/v1`, `/v2`) and deprecation headers.  

- **Cross-Cutting Concerns**  
  - CORS handling for browser-based dashboard clients.  
  - Request/response logging (structured, without sensitive data).  
  - Correlation ID generation and propagation (`X-Correlation-ID`).  

## Security

- **TLS Termination** at the Gateway (or at the load balancer with strict mTLS to Gateway).  
- **API Key Handling**  
  - Keys never logged in plain text; only hashed or truncated representations in logs.  
  - Support for key rotation and revocation.  
- **Input Sanitization** to prevent injection attacks and malformed payloads.
- **CORS** configured per environment and client type (dashboard vs. server-to-server).  

## Observability

- **Logging**  
  - Structured JSON logs with `tenant_id`, `api_key_id` (masked), `method`, `path`, `status`, `correlation_id`.  
  - Redact sensitive headers and bodies according to policy.  

- **Metrics**  
  - `gateway_requests_total{tenant, endpoint, status}`  
  - `gateway_request_duration_seconds{tenant, endpoint}`  
  - `gateway_rate_limit_hits_total{tenant, endpoint}`  
  - `gateway_auth_failures_total{reason}`  

- **Tracing**  
  - Generate or propagate `X-Correlation-ID`.  
  - Integrate with OpenTelemetry for end-to-end traces across Gateway → Ingestion → Workers.  

## Scalability & Performance

- **Stateless design** allows horizontal scaling behind a load balancer.  
- Use non-blocking I/O (e.g., WebFlux) if high concurrency and low latency are critical.  
- Cache frequently accessed data (e.g., API key metadata) with short TTL to reduce DB load.  
- Deploy multiple replicas across availability zones for resilience.  

## Failure Modes & Mitigations

- **Auth service / DB outage**  
  - Cache API key metadata locally with TTL; degrade gracefully (e.g., allow existing valid sessions, block new keys if necessary).  
- **Redis / rate-limit store outage**  
  - Fallback to local in-memory rate limiting with conservative limits.  
- **Internal service unavailable**  
  - Return clear `502 Bad Gateway` / `503 Service Unavailable` with retry-after where applicable.  
- **DDoS / traffic spikes**  
  - Integrate with WAF / edge DDoS protection; enforce strict rate limits and IP-based throttling if needed.  
