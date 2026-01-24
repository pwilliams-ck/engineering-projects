# Technical Decisions

Architecture Decision Records (ADRs) for key choices made during platform development.

---

## 1. Identity Provider: Keycloak over Duo

**Status:** Accepted

**Context:**
Needed an identity provider supporting SAML, OIDC, and LDAP federation for enterprise SSO. Evaluated Keycloak (open source) vs Duo (commercial).

**Decision:** Keycloak

**Rationale:**
- Full protocol support (SAML 2.0, OIDC, LDAP) in a single deployment
- Self-hosted = no per-user licensing costs at scale
- Extensive customization for tenant isolation
- Built-in user federation with Active Directory
- Duo would require additional integration layers for SAML/OIDC

**Tradeoffs:**
- Keycloak requires operational investment (upgrades, HA configuration)
- Duo has better out-of-box MFA UX
- Mitigated by integrating hardware token support directly into Keycloak

---

## 2. Monolith + Saga (Not Microservices)

**Status:** Accepted

**Context:**
Greenfield project with small team (solo engineer, 2 juniors onboarding). ~100 customer operations/day. Needed resilient coordination with external services (identity provider, cloud hypervisor, DR platform).

**Decision:** Stay monolith, add saga orchestration for external service coordination. Don't extract microservices.

**Rationale:**
- The problem is unreliable external services, not internal scaling
- Saga pattern solves that without distribution overhead
- Single deployment unit = simpler operations for small team
- Microservices add: network boundaries, deployment complexity, distributed tracing, service discovery
- None of those solve a problem we actually have
- "If you can't build a well-structured monolith, what makes you think you can build microservices?"

**Evaluated alternatives:**
- **Temporal:** Excellent workflow engine, but requires separate cluster. Overkill for current scale.
- **Microservices:** Adds operational complexity without solving our actual problem.
- **Keep synchronous:** Timeouts and partial failures make this untenable for multi-step provisioning.

**Outcome:**
- Saga orchestrator handles external service failures gracefully
- Single binary deployment, easy for juniors to understand and debug
- Clear upgrade path to Temporal if scale demands it

---

## 3. LDAPS over LDAP

**Status:** Accepted

**Context:**
Integrating with customer Active Directory instances for user federation.

**Decision:** Require LDAPS (LDAP over TLS) for all AD connections. No plaintext LDAP.

**Rationale:**
- LDAP transmits credentials in cleartext (even with SASL, bind passwords are exposed)
- Enterprise customers expect TLS for directory traffic
- Certificate validation prevents MITM attacks
- Minimal implementation difference — just TLS config

**Implementation notes:**
- Accept customer CA certificates for internal PKI
- Validate certificate chains, don't skip verification
- Log certificate expiration warnings

---

## 4. Idempotency Patterns

**Status:** Accepted

**Context:**
Webhooks from CRM can be delivered multiple times. External APIs (cloud hypervisor, identity provider) may timeout without indicating success/failure.

**Decision:** Idempotency keys + at-least-once delivery assumption

**Patterns used:**

**Idempotency keys:**
```go
// Store operation intent before execution
func (s *Service) ProvisionTenant(ctx context.Context, req Request) error {
    key := fmt.Sprintf("provision:%s", req.TenantID)

    // Check if already processed
    if result, exists := s.idempotencyStore.Get(key); exists {
        return result
    }

    // Execute and store result
    result := s.doProvision(ctx, req)
    s.idempotencyStore.Set(key, result, 24*time.Hour)
    return result
}
```

**Idempotent external calls:**
- Use PUT over POST where APIs support it
- Include client-generated request IDs
- Query before create when possible

**Saga compensation:**
- Each step checks current state before acting
- Compensation is also idempotent (delete non-existent = success)

---

## 5. Why Go

**Status:** Accepted

**Context:**
Selecting primary language for backend services.

**Decision:** Go

**Rationale:**

**Operational simplicity:**
- Single static binary, no runtime dependencies
- Cross-compilation trivial
- Small container images (~10-20MB)

**Concurrency model:**
- Goroutines map well to webhook-driven workloads
- Channels for coordination without shared memory bugs
- Context propagation for timeouts/cancellation

**Ecosystem fit:**
- First-class SDKs for hypervisor APIs, Keycloak, cloud providers
- Strong HTTP/REST tooling
- Good LDAP libraries

**Team scalability:**
- Minimal language features = consistent code across team
- Fast onboarding for new engineers
- `gofmt` eliminates style debates

**Considered alternatives:**
- Rust: Higher learning curve, slower iteration
- Python: Deployment complexity, runtime errors
- Java/Kotlin: JVM overhead, slower startup

---

## 6. PostgreSQL as Primary Data Store

**Status:** Accepted

**Context:**
Needed persistent storage for tenant configuration, saga state, and audit logs.

**Decision:** PostgreSQL for everything (no polyglot persistence initially)

**Rationale:**
- JSONB covers semi-structured data needs without separate document store
- ACID transactions critical for saga state consistency
- Mature tooling, extensive documentation
- Single database = simpler operations, backups, monitoring
- Can extract specialized stores later if needed

**Schema approach:**
- Relational tables for core entities
- JSONB columns for flexible/evolving attributes
- Avoid over-normalization early

---

## 7. Nginx as API Gateway

**Status:** Accepted

**Context:**
Need TLS termination, rate limiting, and request routing in front of Go services.

**Decision:** Nginx over dedicated API gateway products

**Rationale:**
- Already in operational toolbox
- Excellent TLS termination performance
- Simple configuration for our routing needs
- No vendor lock-in
- Evaluated Kong, Traefik — added complexity without proportional benefit for our scale

**Configuration approach:**
- TLS termination at Nginx
- Proxy to Go services over localhost
- Rate limiting per endpoint
- Request ID injection for tracing
