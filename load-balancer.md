# Load Balancer — Functionality Checklist

**Purpose:** Distribute traffic across multiple backend instances for availability, scalability, and performance. Protocol-agnostic focus (L4 and/or L7).

## Traffic Distribution
- [ ] Round robin and weighted round robin
- [ ] Least connections (and weighted variant)
- [ ] Least response time
- [ ] Power of two choices
- [ ] IP hash / consistent hashing, with bounded load
- [ ] Zone-aware / locality routing (prefer same-zone backends)
- [ ] Per-request balancing for HTTP/2 and gRPC (not per connection)
- [ ] Session persistence (cookie, header or source IP): off by default, enabled per backend pool
- [ ] Weights adjustable at runtime
- [ ] Slow start for new or recovered backends (prevents thundering herd on recovery)
- [ ] Connection draining on backend removal and deploys (finish in-flight, accept no new)
- [ ] Manual drain/disable of a single backend through the admin API
- [ ] Max connection age and rebalancing of long-lived connections after scale-out
- [ ] Per-backend max connections and request queueing with a queue timeout
- [ ] Backup/standby backends

## Health Checks
- [ ] Active health checks (TCP, HTTP, gRPC; custom path/status/body) against the backend's readiness endpoint
- [ ] Passive health checks / outlier detection: eject on repeated connection failures, timeouts or 5xx
- [ ] Ejection limits: max share of backends ejected, ejection time growing on repeated ejection
- [ ] Configurable thresholds: interval, timeout, healthy/unhealthy counts
- [ ] Flap dampening (hysteresis)
- [ ] Panic threshold: route to all backends when too few are healthy
- [ ] Automatic failover and re-admission of recovered backends
- [ ] Status mapping: 503 no healthy backend, 502 invalid upstream response, 504 timeout
- [ ] LB's own readiness independent of backend health
- [ ] Health and metrics endpoints on a separate internal port

## Retries & Timeouts
- [ ] Retries: off by default, enabled per pool, idempotent methods only, bounded attempts
- [ ] Retry budget (retries capped as a share of requests); connect failures retried first; no retry after the response has started
- [ ] Request hedging: off by default, idempotent requests only
- [ ] Connect, idle and response timeouts, set separately
- [ ] LB idle timeout shorter than the backend keep-alive timeout (prevents random 502s)
- [ ] Timeouts ordered outside-in: LB > backend request > backend's own calls

## Protocol Support
- [ ] L4: TCP, UDP (flow tracking with idle timeout)
- [ ] L7: HTTP/1.1, HTTP/2, WebSocket, gRPC
- [ ] TLS termination and TLS passthrough
- [ ] SNI-based routing
- [ ] Backend re-encryption (TLS/mTLS to upstream)
- [ ] TLS 1.2+ with defined minimum version and cipher policy
- [ ] Certificate management, rotation and reload without restart

## Client Identity
- [ ] Client IP preserved via Proxy Protocol, `X-Forwarded-*` or direct server return (DSR)
- [ ] `X-Forwarded-For` trusted only from known upstream load balancers (right-most trusted hop)
- [ ] Client-supplied forwarding headers stripped before setting its own

## Scalability & Configuration
- [ ] Horizontal scaling of the LB tier
- [ ] Connection pooling / keep-alive to backends
- [ ] Ephemeral port / SNAT exhaustion avoidance: connection reuse, wide port range, multiple source IPs
- [ ] Connection-tracking table sizing
- [ ] Dynamic backend discovery (DNS, service registry, autoscaling groups); DNS TTL respected
- [ ] Autoscaling integration: registration delay, warm-up, deregistration delay
- [ ] Runtime API and hot config reload to add/remove backends without dropping connections
- [ ] Bad config never replaces good; config versioning and signing
- [ ] Declarative config / IaC support
- [ ] Scheduled config export for backup and restore
- [ ] Low-latency, high-throughput data path

## Overload Protection
- [ ] Load shedding when queues or latency pass a threshold
- [ ] Admission control on total connections and requests
- [ ] Bulkheads: limits per backend pool

## Emergency Controls
- [ ] Maintenance mode per pool or listener
- [ ] Instant block of an IP or client through the admin API
- [ ] Drain or disable a whole pool without redeploy

## Dependency Failure Behaviour
- [ ] Service discovery down: last known backends kept
- [ ] DNS failure: last good resolution kept
- [ ] Config source down: running config kept
- [ ] All backends unhealthy: panic routing or 503, per pool
- [ ] Telemetry/log sink down: bounded buffer, requests never blocked

## Availability
- [ ] LB high availability (active-active or active-passive, VRRP/floating IP)
- [ ] Multi-zone deployment
- [ ] Multi-region failover (GSLB/DNS-based)
- [ ] Graceful shutdown: stop new connections, drain in-flight within a limit

## Deployments
- [ ] Blue/green deployments
- [ ] Rolling deployments
- [ ] Canary via weights, with promote and rollback

## Security
- [ ] DDoS protection: SYN-flood mitigation, per-client connection limits, slow-client limits
- [ ] Protocol-attack protection: HTTP/2 rapid reset, header-compression bombs, slowloris and slow-read variants
- [ ] IP allow/deny lists
- [ ] Admin API: authentication, RBAC, separate internal network
- [ ] Backends on a private network reachable only from the LB (network policy)

## Observability
- [ ] Per-backend metrics: active connections, RPS, latency, error rates, health state changes, ejections, shed requests
- [ ] Metric cardinality limits
- [ ] Access and error logs with trace ID and correlation ID; secrets redacted
- [ ] Client IP anonymization per environment; configurable log retention
- [ ] Audit log of config and admin changes
- [ ] Alerts on unhealthy backends and capacity thresholds

## Packaging & Deployment
- [ ] Minimal image, non-root, read-only filesystem
- [ ] Signed images with build provenance; pinned dependency versions
- [ ] CPU/memory requests and limits
- [ ] Time sync (NTP) on hosts

## Documentation
- [ ] Pools that require session affinity, and why
- [ ] Backend responsibility: readiness does not fail on optional dependencies (e.g. a cache-only outage); keep-alive timeout longer than the LB idle timeout
- [ ] Dependency failure matrix
- [ ] Rotation procedures: TLS certificates, mTLS certificates, admin credentials
- [ ] Disaster recovery: RTO/RPO, config backup and restore procedure
- [ ] Capacity and sizing per instance: connections, throughput, memory
- [ ] Privacy: log retention periods, IP anonymization policy
- [ ] Vulnerability (CVE) patch response times
- [ ] Runbooks: backend down, all backends unhealthy, LB failover, maintenance mode, pool drain
