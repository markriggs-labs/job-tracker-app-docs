# ADR-004: PostgreSQL as the primary relational database

**Status:** Accepted  
**Date:** 2026-04-11  
**Author:** Mark

## Context

The application manages structured relational data including job requisitions, contacts, activity journals, and user preferences. Relationships between entities are well-defined and querying across them will be common. A decision is required on the primary data store.

## Decision

PostgreSQL will be used as the primary relational database. Each microservice will own its own schema within the database, enforcing logical data separation while keeping operational complexity manageable for a portfolio project. Schema migrations are managed per service using Entity Framework Core.

## Rationale

- The Job Tracker data model is inherently relational — job reqs link to contacts, contacts link to activities, all records scope to a user.
- PostgreSQL is the most widely used open-source relational database and is well-supported in containerized environments.
- Full SQL support enables complex queries for reporting and dashboard features in future phases.
- Schema-per-service provides logical domain separation without the operational overhead of running a separate database instance per service.
- PostgreSQL has excellent support for JSON columns, which provides flexibility for semi-structured data like activity notes and AI-generated content.

## Alternatives considered

- **MongoDB:** Document model is a good fit for activity journals but less ideal for the relational aspects of the data model.
- **MySQL:** A solid alternative but PostgreSQL has broader feature support and stronger JSON capabilities.
- **Separate database per service:** Ideal in a true microservices production environment but adds significant operational overhead for a portfolio project.

## Consequences

- Schema migrations are managed per service using Entity Framework Core migrations.
- Connection pooling should be considered as service count grows.
- A schema-per-service approach can be migrated to separate databases in the future if needed without application-level changes.
- **EF Core `Database.MigrateAsync()` runs on startup in all environments, including Production.** Migrations are idempotent by design — EF Core tracks applied migrations in the `__EFMigrationsHistory` table and skips any that have already been applied. This removes the need for a separate migration step during deployment at the cost of a slightly longer cold-start. Migrations must be verified locally before deploying; a failed migration on startup will crash the service.
