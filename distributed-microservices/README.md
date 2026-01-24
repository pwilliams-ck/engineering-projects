# Distributed Microservices Architecture

## Overview

Production-grade distributed microservices system built with Go, demonstrating enterprise patterns for service communication, authentication, logging, and event-driven architecture.

**Repository:** [github.com/pwilliams-ck/distributed-microservices](https://github.com/pwilliams-ck/distributed-microservices)

---

## Why I Built This

After working on monolithic systems, I wanted to understand *why* software gets built as microservices. The industry talks about microservices constantly, but the reasoning often felt hand-wavy.

Questions I wanted to answer:
- When do multiple protocols actually matter?
- What problems does database-per-service solve?
- Is the complexity worth it?

---

## The Solution

Built a complete microservices ecosystem with four distinct communication protocols, showing when and why to use each.

```
┌─────────────────┐
│   Front-End     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Broker Service │ ◄─── API Gateway
└────────┬────────┘
         │
    ┌────┴────┬──────────┬──────────┐
    ▼         ▼          ▼          ▼
┌───────┐ ┌───────┐ ┌───────┐ ┌──────────┐
│ Auth  │ │Logger │ │ Mail  │ │ Listener │
│(HTTP) │ │(gRPC) │ │(HTTP) │ │(RabbitMQ)│
└───┬───┘ └───┬───┘ └───┬───┘ └────┬─────┘
    ▼         ▼         ▼          ▼
┌───────┐ ┌───────┐ ┌───────┐ ┌──────────┐
│Postgres│ │MongoDB│ │MailHog│ │ RabbitMQ │
└───────┘ └───────┘ └───────┘ └──────────┘
```

---

## Communication Patterns

### When to Use Each Protocol

| Protocol | Use Case | Trade-off |
|----------|----------|-----------|
| **HTTP/REST** | External APIs, debugging, simple CRUD | Universal but slower |
| **RPC** | Internal Go-to-Go calls | Fast but Go-only |
| **gRPC** | High-performance, polyglot services | Fast, typed, but complex setup |
| **RabbitMQ** | Async operations, decoupling, events | Resilient but eventual consistency |

### Example: Logger Service

The logger demonstrates all patterns — clients can write logs via HTTP, RPC, or gRPC, while the listener service pushes logs asynchronously through RabbitMQ.

```go
// gRPC — High-performance logging
client := logs.NewLogServiceClient(conn)
client.WriteLog(ctx, &logs.LogRequest{
    LogEntry: &logs.Log{Name: "event", Data: "message"},
})

// RabbitMQ — Fire-and-forget async
ch.Publish("logs_topic", "log.INFO", false, false,
    amqp.Publishing{Body: jsonData})
```

---

## Core Services

| Service | Port | Database | Purpose |
|---------|------|----------|---------|
| Broker | 8080 | — | API gateway, request routing |
| Auth | 8081 | PostgreSQL | User authentication, bcrypt |
| Logger | 8082 | MongoDB | Centralized logging |
| Mail | 8083 | — | SMTP with templates |
| Listener | — | — | RabbitMQ consumer |

---

## Technology Stack

**Backend:** Go 1.18, Chi Router, gRPC, Protocol Buffers

**Databases:** PostgreSQL (users), MongoDB (logs)

**Infrastructure:** Docker, Docker Compose, RabbitMQ, MailHog

---

## Key Design Decisions

### Why Multiple Protocols?

Real distributed systems rarely use just one communication pattern. This project demonstrates protocol selection based on requirements:

- **Auth uses HTTP** — Needs to be debuggable, called infrequently
- **Logger supports all** — High-volume writes benefit from gRPC; async events use RabbitMQ
- **Listener uses AMQP** — Pure event consumer, no synchronous API needed

### Why Database-per-Service?

Each service owns its data:
- Auth service owns PostgreSQL user table
- Logger service owns MongoDB logs collection
- No shared database coupling between services

### Why Broker as Gateway?

Single entry point simplifies:
- Client configuration (one URL)
- Cross-cutting concerns (auth, logging, rate limiting)
- Protocol translation (HTTP in, gRPC/RPC/AMQP out)

---

## Running Locally

```bash
git clone https://github.com/pwilliams-ck/distributed-microservices
cd distributed-microservices/project
make up_build
```

**Access points:**
- Front-end UI: http://localhost:8082
- Broker API: http://localhost:8080
- RabbitMQ Management: http://localhost:15672
- MailHog: http://localhost:8025

---

## What I Learned

The biggest insight: **microservices solve organizational problems more than technical ones.**

The technical benefits are real — independent deployment, protocol flexibility, isolated failures. But a monolith can do most of this with good module boundaries.

What microservices actually solve:
- **Team autonomy** — Auth team deploys without coordinating with logging team
- **Language flexibility** — One team uses Go, another uses Python, nobody argues
- **Ownership clarity** — Service boundaries map to team boundaries
- **Independent scaling** — Not of compute, but of *people* working on the codebase

For a solo developer or small team, a well-structured monolith is almost always simpler. Microservices start making sense when coordination costs between teams exceed the complexity costs of distribution.

### Technical takeaways

1. **Protocol selection matters** — gRPC is faster than HTTP for internal calls, but HTTP is easier to debug
2. **Message queues add resilience** — RabbitMQ decouples services; failures don't cascade
3. **Database-per-service forces good boundaries** — If you can't split the database, you probably can't split the service
4. **Docker Compose is underrated** — Full distributed system running locally with one command
