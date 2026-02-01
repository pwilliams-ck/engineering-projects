# Engineering Projects

Documentation of production systems and a few side projects I've built. Mostly
Go backend work, some hardware.

## Contents

- [Production Work](#production-work)
  - [Multi-Tenant Cloud Platform](#multi-tenant-cloud-platform)
  - [Pi Pico Environmental Sensors](#pi-pico-environmental-sensors)
- [Side Projects](#side-projects)
  - [Distributed Microservices](#distributed-microservices)

---

## Production Work

### [Multi-Tenant Cloud Platform](./cloud-platform/)

Designed and built backend automation for an enterprise cloud platform.
Currently architecting migration from synchronous request handling to saga-based
orchestration for resilience.

**Tech:** Go, PostgreSQL, Keycloak, on-prem hypervisor

**Highlights:**

- Solo backend engineer, end-to-end ownership
- SSO/MFA integration (SAML, OIDC, LDAP)
- Saga pattern for fault-tolerant provisioning
- [Architecture decisions](./cloud-platform/decisions.md) ・
  [Code samples](./cloud-platform/code-samples.md)

### [Pi Pico Environmental Sensors](./pico-sensor/)

Low-cost IoT monitoring system built with Raspberry Pi Pico W. Running 3+ years
continuously through extreme temperatures with zero downtime.

**Tech:** MicroPython, Raspberry Pi Pico W

**Highlights:**

- Total hardware cost: <$20
- Zero maintenance since deployment
- [View source](https://github.com/pwilliams-ck/pico-sensor)

---

## Side Projects

### [Distributed Microservices](./distributed-microservices/)

Production-grade microservices architecture in Go demonstrating multiple
communication patterns (HTTP, RPC, gRPC, RabbitMQ) with API gateway,
authentication, and event-driven design.

**Tech:** Go, PostgreSQL, MongoDB, RabbitMQ, Docker, gRPC

**Highlights:**

- Four distinct communication protocols with real use cases
- Database-per-service isolation
- Full Docker Compose orchestration
- [View source](https://github.com/pwilliams-ck/distributed-microservices)
