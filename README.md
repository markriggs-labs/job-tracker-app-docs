# job-tracker-app-docs

Documentation repository for the Job Tracker application. Contains architecture decision records, user stories, system diagrams, and developer guides.

## Contents

| Folder | Contents |
|--------|----------|
| `adr/` | Architecture Decision Records |
| `user-stories/` | User stories by domain |
| `architecture/` | EA artifacts — reference architecture, technology roadmap, engineering standards, security architecture, and executive brief |
| `briefs/` | Project briefs |
| `guides/` | Developer setup and contribution guides |

## Documents

### Architecture Documents
- [Reference Architecture — C4 System Context + Container Diagram](architecture/reference-architecture.md)
- [Technology Roadmap](architecture/technology-roadmap.md)
- [Engineering Standards & Governance](architecture/engineering-standards-governance.md)
- [Security Architecture](architecture/security-architecture.md)
- [Executive Architecture Brief](architecture/executive-architecture-brief.md)

### Architecture Decision Records
- [ADR-001: Microservices with Docker and Kubernetes](adr/ADR-001-microservices-architecture.md)
- [ADR-002: Kafka as async work queue](adr/ADR-002-kafka-messaging.md)
- [ADR-003: Auth0 for identity management](adr/ADR-003-auth0-identity.md)
- [ADR-004: PostgreSQL as primary data store](adr/ADR-004-postgresql.md)
- [ADR-005: Blob storage for resume files](adr/ADR-005-blob-storage.md)
- [ADR-006: React and Angular frontends](adr/ADR-006-frontend-frameworks.md)
- [ADR-007: SignalR transport selection — LongPolling over WebSocket](adr/ADR-007-signalr-transport.md)
- [ADR-008: AI integration strategy — dual-model approach with server-side prompt assembly](adr/ADR-008-ai-integration.md)
- [ADR-009: Terraform for VPS infrastructure provisioning](adr/ADR-009-terraform-vps-provisioning.md)
- [ADR-010: Testing strategy](adr/ADR-010-testing-strategy.md)
- [ADR-011: YARP Reverse Proxy as the API Gateway](adr/ADR-011-yarp-api-gateway.md)
- [ADR-012: Nginx Proxy Manager for TLS termination](adr/ADR-012-nginx-proxy-manager-tls.md)

### User Stories
- [All user stories](user-stories/user-stories.md)

### Guides
- [Local development setup](guides/local-development.md)
- [Repository overview](guides/repository-overview.md)

## Future Enhancements

### Observability
- **Structured logging** — Integrate Serilog across all microservices to produce JSON-formatted logs that can be shipped to a centralized logging platform such as Azure Monitor, AWS CloudWatch, or the ELK stack.
- **Metrics** — Add Prometheus metrics endpoints to each service and a Grafana dashboard to visualize API request rates, error rates, and Kafka publish success/failure counts.
- **Distributed tracing** — Implement OpenTelemetry tracing to track requests as they flow across services, with visualization via Jaeger or Azure Application Insights.
- **Dead letter queue** — Route failed Kafka publish attempts to a dead letter topic for retry and investigation rather than silently dropping events.

### Security
- **File type validation and virus scanning** — Add file type enforcement and malware scanning to the Resume Service before files are written to blob storage.
- **Rate limiting** — Add request rate limiting at the API gateway level to prevent abuse.

### Frontend
- **Angular implementation** — Build the Angular frontend consuming the same API gateway as the React frontend, demonstrating frontend-agnostic backend design.
- **Reporting dashboard** — Phase 3 pipeline view showing requisitions by status, activity trends, and response rates by source.

### AI Features
All AI features are user-triggered via explicit action — analysis is never fired automatically on save. This gives the user control over when content is ready for analysis and avoids burning API credits on in-progress drafts.

- **Job description analysis** — User triggers analysis from the job detail page; AI service returns insights, key requirements, and skills extracted from the stored job description.
- **Interview prep question generation** — Generate likely interview questions from a stored job description using the AI service.
- **Job scoring** — Score a requisition against user-defined target role criteria and return a match percentage with rationale.
