# Engineering Projects

Software Engineer — Backend systems, identity & access management, cloud
infrastructure. Go.

## Contents

- [Production Work](#production-work)
  - [Multi-Tenant Cloud Platform](#multi-tenant-cloud-platform)
- [Learning & Exploration](#learning-and-exploration)
  - [Distributed Microservices](#distributed-microservices)

---

## Production Work

### [Multi-Tenant Cloud Platform](./cloud-platform/)

Backend automation platform I solely architected and engineered. Enterprise
identity federation (SAML, SCIM, LDAPS), cloud provisioning, and disaster
recovery through a unified Go orchestration system. Serves 50+ enterprise
clients.

**Tech:** Go, PostgreSQL, Keycloak, on-prem hypervisor

**Highlights:**

- Solo backend engineer, end-to-end ownership
- SSO/MFA integration (SAML, OIDC, LDAP)
- Orchestration pattern for fault-tolerant provisioning

-> [Architecture decisions](./cloud-platform/decisions.md)

-> [Code samples](./cloud-platform/code-samples.md)

---

## Learning and Exploration

### [Distributed Microservices](./distributed-microservices/)

Go microservices ecosystem exploring HTTP, RPC, gRPC, and event-driven
communication patterns. Built following Trevor Sawler's course to understand
when distributed architecture is and isn't warranted.

**Tech:** Go, PostgreSQL, MongoDB, RabbitMQ, Docker, gRPC

**Highlights:**

- Four distinct communication protocols with real use cases
- Database-per-service isolation
- Full Docker Compose orchestration

-> [View source](https://github.com/pwilliams-ck/distributed-microservices)
