# Reverse Proxy — Functionality Checklist

**Purpose:** Sits in front of one or more servers, forwarding client requests to them. Hides origin servers and offloads TLS, caching, and compression. Generic L7 (sometimes L4), not API-aware.

## Core Proxying
- [ ] Forward requests to upstream(s) by host, path, port
- [ ] Virtual hosts / multiple sites on one IP; explicit hosts, wildcard hosts in Development only
- [ ] HTTP/1.1, HTTP/2, HTTP/3 (with `Alt-Svc` advertisement), WebSocket, gRPC, streaming
- [ ] Explicit allowed methods per route (405 otherwise)
- [ ] Public listener routes only the public path prefix; health, metrics and docs on an internal listener
- [ ] Redirects and URL rewriting per route; paths forwarded unchanged by default
- [ ] `Location`, `Refresh` and `Set-Cookie` domain/path rewritten when a route rewrites paths or hosts
- [ ] Host header preserved or rewritten per route
- [ ] Hop-by-hop header handling
- [ ] Request/response header add/remove/modify
- [ ] Upstream connection pool: max connections per server, keep-alive, connection lifetime
- [ ] Request mirroring/shadowing to a second upstream (responses discarded)
- [ ] Geo/IP-based routing and blocking

## Request Normalization
- [ ] Path canonicalized before routing and access rules: `../`, `./`, duplicate slashes, trailing slashes
- [ ] Double/overlong percent-encoding rejected
- [ ] Null bytes and control characters rejected
- [ ] `Host` checked against the allowed-hosts list; `X-Forwarded-Host` validated the same way
- [ ] Open-redirect protection in redirects and rewrites

## Headers & Correlation
- [ ] Client-supplied `X-Forwarded-*` and internal headers (`X-Tenant-Id`, `X-User-Id`, `X-Internal-*`) stripped before forwarding
- [ ] `X-Forwarded-For/Proto/Host` and `Forwarded` set by the proxy
- [ ] Multi-hop trusted proxy chain: `X-Forwarded-For` parsed from the right-most trusted hop
- [ ] Correlation ID created or validated; trace headers propagated and returned on every response, including proxy-made errors
- [ ] `Server`, `X-Powered-By` and other stack-revealing headers removed

## Authentication
- [ ] Basic auth / simple access control
- [ ] JWT / OIDC validation
- [ ] Forward-auth: auth subrequest to an external service before proxying
- [ ] Client certificate (mTLS) authentication

## Timeouts & Retries
- [ ] Connect, idle/activity and response timeouts, separate from the route timeout
- [ ] Timeouts ordered outside-in: proxy > upstream request > upstream's own calls
- [ ] Streaming routes: longer timeout, buffering off
- [ ] Retries: off by default, enabled per route, idempotent methods only, bounded attempts
- [ ] Retry budget (retries capped as a share of requests); no retry after the response has started

## Limits
- [ ] Header size/count, connections, HTTP/2 streams, minimum request data rate
- [ ] Global body size cap with per-route overrides; request buffering control
- [ ] Maximum upstream response size and response header size
- [ ] `Expect: 100-continue` handled; body rejected before upload when limits fail
- [ ] Status codes: 413 body, 414 URL, 431 headers

## TLS
- [ ] TLS termination
- [ ] TLS 1.2+ with defined minimum version and cipher policy
- [ ] HTTP → HTTPS redirect (308) and HSTS
- [ ] Automatic cert issuance/renewal (e.g. ACME) and reload without restart
- [ ] SNI multi-domain certificates
- [ ] OCSP stapling
- [ ] TLS / mTLS to upstream

## Caching
- [ ] Response caching with TTL, purge/invalidation
- [ ] Cache key design; honours `Cache-Control` and `Vary`
- [ ] Authenticated responses cached only when the user is part of the key
- [ ] Cache poisoning protection: unkeyed headers ignored or normalized
- [ ] Stampede protection: one fetch per key (cache lock), stale served while refreshing
- [ ] Negative caching (404) with a short TTL
- [ ] Stale-cache serving on upstream failure
- [ ] Authenticated purge on the internal listener

## Static Files & Compression
- [ ] Static file serving
- [ ] No directory listing; dotfiles denied; path traversal blocked
- [ ] Correct MIME types with `X-Content-Type-Options: nosniff`
- [ ] Compression (gzip, brotli) with minimum size and content-type list
- [ ] Precompressed assets served when present
- [ ] Compression disabled per route where responses mix secrets with attacker-controlled input (BREACH)
- [ ] Conditional requests (ETag, If-Modified-Since)
- [ ] Range request support
- [ ] Slow-client buffering to protect upstreams

## Security
- [ ] Origin IP hiding; upstreams on a private network reachable only from the proxy (network policy)
- [ ] Rate limiting and connection limits
- [ ] IP allow/deny lists, globally and per route
- [ ] Security headers (CSP, X-Frame-Options, X-Content-Type-Options, Referrer-Policy, etc.)
- [ ] Request smuggling and malformed-request protection
- [ ] Protocol-attack protection:
  - HTTP/2 rapid reset (stream reset rate limit)
  - HPACK/QPACK header-compression bombs
  - slowloris and slow-read variants
  - chunked encoding and trailer abuse
  - h2c upgrade smuggling blocked
- [ ] WAF integration

## Overload Protection
- [ ] Load shedding when queues or latency pass a threshold
- [ ] Bounded request queues (backpressure)
- [ ] Per-upstream connection and concurrency limits (bulkheads)

## Emergency Controls
- [ ] Maintenance page, global or per route, with `Retry-After`
- [ ] Kill switch per route without redeploy
- [ ] Instant block of an IP or client

## Dependency Failure Behaviour
- [ ] DNS failure: last good resolution kept; TTL respected otherwise
- [ ] Config source down: running config kept
- [ ] Cache storage full or unavailable: requests bypass the cache
- [ ] Telemetry/log sink down: bounded buffer, requests never blocked

## Reliability
- [ ] Multiple upstreams with basic load balancing
- [ ] Active and passive health checks, with failover
- [ ] Own `/health/live` and `/health/ready`, independent of upstreams, on an internal port
- [ ] Graceful reload and shutdown: drain in-flight within a limit, flush telemetry

## Errors
- [ ] Proxy-made errors: problem details (`title`, `detail`, stable `code`, `traceId`) on API routes and JSON `Accept`; custom HTML error pages on browser routes
- [ ] Status map: 404 no route, 405, 413, 414, 431, 429, 502, 503, 504
- [ ] Upstream errors passed through unchanged by default; upstream error interception (replace with own page) per route
- [ ] No internal hostnames or exception text in errors

## Observability
- [ ] Access and error logs (configurable format)
- [ ] Body logging off by default; opt-in per route with size limit
- [ ] Redaction of `Authorization`, cookies and tokens in logs
- [ ] Client IP anonymization per environment; configurable log retention
- [ ] Metrics: requests, status codes, latency, cache hit ratio, shed requests
- [ ] Metric cardinality limits
- [ ] Request ID injection for tracing
- [ ] Audit log of config and admin changes

## Operations
- [ ] Simple text/declarative config
- [ ] Config test/validation before reload; bad config never replaces good
- [ ] Config versioning; scheduled config export for backup and restore
- [ ] Reload without dropping connections
- [ ] Admin/reload interface: authentication, RBAC, internal listener only
- [ ] Low resource footprint
- [ ] Log rotation

## Packaging & Deployment
- [ ] Minimal image, non-root, read-only filesystem
- [ ] Signed images with build provenance; pinned dependency versions
- [ ] Secrets and certificates from a secret store
- [ ] CPU/memory requests and limits
- [ ] Time sync (NTP) on hosts

## Documentation
- [ ] Upstream responsibility: trust forwarded headers only from the proxy's network
- [ ] Dependency failure matrix
- [ ] Rotation procedures: TLS certificates, mTLS certificates, admin credentials
- [ ] Disaster recovery: RTO/RPO, config backup and restore procedure
- [ ] Capacity and sizing per instance: connections, throughput, memory
- [ ] Privacy: log retention periods, IP anonymization policy
- [ ] Vulnerability (CVE) patch response times
- [ ] Runbooks: upstream down, maintenance mode, cache purge, certificate renewal failure
