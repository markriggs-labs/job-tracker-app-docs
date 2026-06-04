# ADR-002: Kafka as an async work queue for job application events

**Status:** Accepted  
**Date:** 2026-04-11  
**Author:** Mark  
**Last updated:** 2026-05-10

## Context

Services need to communicate with each other without tight coupling. The job service handles create and edit operations for job requisitions. Rather than processing all downstream work synchronously in the request path, a decision is required on how to offload that work so the API can respond immediately and consumers process independently.

## Decision

Apache Kafka will be used as an async work queue. The job service publishes a full job application payload to Kafka on create (`job.application.created`) and edit (`job.application.updated`) and returns **202 Accepted** immediately — it does not write to the database. The Job Service Consumer subscribes to both topics, writes the record to the database via EF Core, and pushes a real-time SignalR event to the React client so it can invalidate its local query cache.

Status changes are handled synchronously — they are lightweight field updates that do not require downstream processing and remain in the Job Service with a direct DB write.

AI analysis is never triggered automatically. It is user-initiated via an explicit action on the job detail page, so AI-related events are not part of the Kafka pipeline.

**Contacts, journal entries, and resumes are handled synchronously** and do not publish to Kafka. See the decision boundary section below.

## Rationale

- The job service does not need to know that any consumer exists. Producers and consumers are fully decoupled.
- Publishing the full job payload (all database fields) to Kafka means any consumer has everything it needs without calling back to the job service.
- Kafka topics act as a durable event log, enabling event replay and audit history.
- New consumers can be added to existing topics without modifying the producer, supporting future extensibility. The AI service, for example, can subscribe to `job.application.created` to pre-process job descriptions without any changes to the job service.
- Under high load, consumers process at their own pace without the API blocking or dropping requests.
- Kafka is the industry standard for distributed event streaming and demonstrates enterprise-grade messaging architecture.
- SignalR push from the consumer closes the loop to the React client — the UI refreshes when the DB write completes rather than relying on polling or optimistic updates.

## Decision boundary: when to use Kafka vs synchronous writes

The guiding principle is **fan-out**. Kafka is justified when an event needs to reach more than one consumer, or when the producing service should be decoupled from the side effects of its own writes. It is not justified simply because data is being written.

| Service | Pattern | Reason |
|---------|---------|--------|
| Job Service (create/edit) | Async via Kafka | Multiple consumers: DB writer, notification service, future AI service. The job service must not know who processes its events. |
| Job Service (status change) | Synchronous | Single lightweight field update. No downstream service subscribes to status changes. |
| Contact Service | Synchronous | Single consumer (itself). No other service needs to react to a contact being added or edited. |
| Journal Service | Synchronous | Single consumer (itself). Journal entries are a private per-job audit trail with no cross-service subscribers. |
| Resume Service | Synchronous | Single consumer (itself). Binary file payloads (resumes, cover letters) are not appropriate for Kafka regardless; metadata writes have no cross-service subscribers. |
| AI Service (future) | Synchronous / user-triggered | Analysis is initiated by the user, not by a system event. No fan-out required. |
| Feedback (user-submitted) | Async via Kafka | The API returns 202 immediately; the Notification Service consumes `feedback.submitted` and sends an email. Decouples the HTTP response from the SMTP call. |

Adding Kafka to a service with a single consumer introduces broker dependency, consumer group management, and topic pre-creation overhead with no architectural benefit. The contact, journal, and resume services gain nothing from async processing today. If a future requirement introduces a second consumer for those events (e.g., a notification on contact interaction, or AI analysis of journal entries), the decision should be revisited and those services migrated to the Kafka pattern at that point.

## Alternatives considered

- **REST-based synchronous calls:** Introduces tight coupling between services and cascading failure risk.
- **RabbitMQ:** A solid alternative message broker, but Kafka's log-based model and scalability story are stronger for portfolio demonstration.
- **In-process events:** Only viable within a monolith, not applicable here.
- **Polling for DB write completion:** Simple but chatty and adds latency. SignalR push is more efficient and demonstrates real-time capability.

## Consequences

- Requires a running Kafka instance in the local development environment via Docker Compose.
- Job Service Consumer is a separate service that owns the DB write for create/edit — the Job Service is publisher-only for these operations.
- Job Service Consumer hosts the SignalR hub; the API Gateway proxies WebSocket connections to it.
- Teams need familiarity with Kafka consumer group semantics.
- Enables future features like event sourcing and audit logging with minimal additional work.

## Kafka topics

| Topic | Publisher | Trigger | Payload |
|-------|-----------|---------|---------|
| `job.application.created` | Job Service | POST /api/jobs | JobReqId, UserId, UserEmail, CompanyName, RoleTitle, SourceUrl, CompanyCareerPortalUrl, JobDescription, DateDiscovered, ApplicationExpiryDate, InterviewDate, OccurredAt |
| `job.application.updated` | Job Service | PUT /api/jobs/{id} | JobReqId, UserId, UserEmail, CompanyName, RoleTitle, SourceUrl, CompanyCareerPortalUrl, JobDescription, DateDiscovered, ApplicationExpiryDate, DateSubmitted, InterviewDate, OccurredAt |
| `feedback.submitted` | Notification Service | POST /api/feedback | UserId, UserEmail, UserName, Content, OccurredAt |

## Current consumers

| Consumer | Topic | Action |
|----------|-------|--------|
| Job Service Consumer | `job.application.created` | Writes new job requisition to DB; pushes `jobCreated` SignalR event |
| Job Service Consumer | `job.application.updated` | Updates job requisition in DB; pushes `jobUpdated` SignalR event |
| Notification Service | `job.application.created` | Logs event |
| Notification Service | `job.application.updated` | Logs event |
| Notification Service | `feedback.submitted` | Sends feedback email to configured recipient via MailKit SMTP |

## Topic management

Topics are pre-created by a `kafka-init` one-shot service in `docker-compose.yml`. This service runs after the Kafka broker passes its healthcheck and exits once both topics exist. All Kafka-consuming services (`job-service`, `job-service-consumer`, `notification-service`) declare a `depends_on` condition of `service_completed_successfully` on `kafka-init`, guaranteeing topics are present before any consumer attempts to subscribe.

This avoids two failure modes encountered during development:

- **Broker-side auto-create disabled:** The Confluent cp-kafka image defaults `auto.create.topics.enable=false`, which overrides the client-side `AllowAutoCreateTopics = true` setting. Topics must exist before a consumer subscribes or the broker returns `Unknown topic or partition`.
- **Race condition on fresh environments:** Without explicit topic pre-creation, the first consumer to start on an empty broker would fail its initial `Consume()` call and enter a retry loop, causing confusing startup errors.

The `--if-not-exists` flag on each `kafka-topics --create` call makes the init step idempotent — safe to run on restarts where topics already exist.
