# ADR-001: Microservices architecture with Docker and Kubernetes

**Status:** Accepted  
**Date:** 2026-04-11  
**Author:** Mark

## Context

The Job Tracker application needs to support multiple domains — job requisitions, contacts, activity journaling, resume management, notifications, and AI-assisted features. A decision is required on whether to build this as a monolith or decompose it into independently deployable services.

## Decision

The system will be built as a set of microservices, each owning its own domain. Services will be containerized using Docker and orchestrated with Kubernetes.

## Rationale

- Each service can be scaled independently based on load. The notification service, for example, may spike during reminder batches without requiring the entire application to scale.
- Services can be developed, tested, and deployed independently, reducing risk per deployment.
- Domain boundaries are enforced by service boundaries, keeping codebases focused and maintainable.
- Kubernetes provides automatic restarts, rolling deployments, and health checks out of the box.
- The architecture directly mirrors real-world enterprise patterns used in healthcare and financial services domains.
- Docker Compose provides a straightforward local development environment before Kubernetes is needed.

## Alternatives considered

- **Monolithic architecture:** Simpler to build initially but difficult to scale specific domains and introduces higher coupling over time.
- **Serverless functions:** Low operational overhead but not ideal for demonstrating container orchestration and inter-service communication patterns.

## Consequences

- Increased initial complexity compared to a monolith.
- Requires a service discovery and inter-service communication strategy — addressed by the YARP API Gateway (see ADR-011).
- Local development requires Docker Compose configuration to run all services together.
- Demonstrates Kubernetes orchestration, which is a key portfolio talking point.
- **Docker Compose is the current production runtime.** The full application stack runs via `docker compose up` on a single Akamai VPS. Kubernetes remains the target orchestration platform for Phase 2, when horizontal scaling or multi-node deployment becomes a requirement.
- **Polyrepo structure:** Each microservice, frontend, and supporting library has its own git repository (14 repositories total). This enforces deployment independence and prevents unintended cross-service coupling at the dependency level. The VS Code workspace file in `job-tracker-app-workspace` references all repos for a unified development experience.
