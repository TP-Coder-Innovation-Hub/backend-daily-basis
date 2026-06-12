# Backend Daily Basis

For backend devs, by backend devs. Assumes Java + Spring Boot knowledge.

This course draws from real production experience as a Java/Spring developer. Spring Boot is the primary vehicle because it's what I know best — the patterns and concepts (messaging, observability, resilience, concurrency) are universal and apply to any backend stack. When other languages are the right tool (TypeScript for serverless, Kotlin for reactive, Scala for actors), we use them directly.

Each topic is short, direct, and packed with working code examples.

## Objectives

- Master REST, gRPC, GraphQL API design and implementation
- Configure applications for any environment using modern practices
- Optimize database access, caching, and storage
- Build asynchronous systems with queues, streams, and scheduling
- Handle concurrency with thread pools, reactive, and actor models
- Implement advanced architectures: BFF, event sourcing, CQRS, sagas
- Secure APIs with OAuth2, JWT, API keys, and OWASP practices
- Operate in production with CI/CD, observability, and resilience patterns

## Course Modules

| # | Module | Topics |
|---|--------|--------|
| 01 | [API and Business Logic](01-api-and-business-logic/) | REST CRUD, gRPC, GraphQL, OpenAPI, Batch, LLM, Icons, Data Platform |
| 02 | [Configuration](02-configuration/) | Environment Variables, Config Files, Vault Injection |
| 03 | [Databases](03-databases/) | Caching, Object Storage, Query Optimization, Batch vs One-by-One |
| 04 | [Asynchronous](04-asynchronous/) | WebSocket, Cronjob Scheduling, Message Queue, Pub/Sub, Streaming |
| 05 | [Concurrency](05-concurrency/) | Thread Pool, Reactive Workload (Kotlin + Java), Actor Workload (Scala + Akka) |
| 06 | [Advanced Architectures](06-advanced-architectures/) | BFF, API Gateway, Serverless (TypeScript + Python), Virtual Queue, Event Sourcing, CQRS, Saga, Orchestrator, Choreography, Service Mesh |
| 07 | [Security](07-security/) | Auth, User Login, API Key, OAuth2 Backend, OWASP |
| 08 | [DevOps](08-devops/) | CI/CD Jenkins, CI/CD GitHub Actions, Distributed Config, Circuit Breaker, Retry, Rate Limit, Timeout, Load Testing, Metrics, Logging, Tracing, Observability Strategy |
