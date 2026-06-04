# ADR-011: YARP Reverse Proxy as the API Gateway

**Status:** Accepted  
**Date:** 2026-04-15  
**Author:** Mark

## Context

With multiple microservices running on separate ports, the React frontend would otherwise need to know the address of every service it calls, handle per-service CORS configuration, and validate Auth0 JWTs independently in each service. A decision is required on whether to introduce an API gateway layer and, if so, which technology to use.

## Decision

A dedicated API Gateway service built on **Microsoft YARP (Yet Another Reverse Proxy)** is used as the single entry point for all client traffic. YARP runs as a .NET 8 service on port 5000 locally and port 8080 in Docker. It is responsible for:

- Validating Auth0 JWTs on every inbound request before forwarding downstream
- Routing requests to the correct microservice based on path prefix
- Proxying SignalR WebSocket connections (with `app.UseWebSockets()` in the middleware pipeline)
- Enforcing CORS policy, with allowed origins driven by the `Cors:AllowedOrigins` environment variable

All YARP routing is declared in `appsettings.json` (and `appsettings.Production.json`) rather than in code, making route changes a configuration-only operation with no recompile required.

## Rationale

- **Single origin for the frontend:** The React app communicates exclusively with the gateway via `VITE_GATEWAY_URL`. No individual service URL ever reaches the browser, and the internal service topology is fully hidden from clients.
- **Centralised JWT validation:** Auth0 token verification is enforced once at the boundary. Downstream services trust the forwarded request without redundant token validation logic in every service.
- **Native .NET library:** YARP is an in-process library, not a sidecar or separate container. The gateway is a standard .NET 8 project — observable, debuggable, and deployable the same way as every other service in the stack.
- **Config-driven routing:** Adding a new service requires a new cluster and route entry in `appsettings.json`. No code changes, no recompile.
- **Portfolio signal:** An explicit API gateway layer demonstrates understanding of the API Gateway pattern, a standard enterprise architecture component.

## Alternatives considered

- **Ocelot:** The established .NET API gateway library. More opinionated and heavier than YARP; requires its own JSON config file format. YARP integrates more naturally with the standard .NET middleware pipeline and `appsettings.json`.
- **nginx as a reverse proxy:** Would require a separate container and nginx config syntax. Adds a non-.NET component to the stack with no meaningful benefit over YARP for this use case.
- **No gateway (direct service calls from frontend):** Each service would need its own CORS config, its own JWT validation middleware, and its own port exposed to the browser. Replicates infrastructure work across every service and exposes the internal service topology to the client.
- **AWS API Gateway / Azure API Management:** Managed cloud gateways with advanced features (rate limiting, analytics, developer portal). Significant cost and operational overhead for a self-hosted portfolio application.

## Consequences

- All inbound client traffic flows through a single point — the gateway is a critical path component. A gateway restart briefly disrupts all client connectivity.
- The YARP middleware pipeline includes `app.UseWebSockets()` to support SignalR WebSocket proxying. In production, the gateway sits behind Nginx Proxy Manager (TLS termination), forming a double-hop proxy chain (Browser → NPM → YARP → Kestrel). This chain prevented reliable WebSocket frame forwarding and directly drove the SignalR LongPolling decision documented in ADR-007.
- CORS allowed origins are driven by the `Cors:AllowedOrigins` environment variable, so adding a new frontend origin (e.g., GitHub Pages URL after a GitHub Organization migration) requires only an environment variable update — no code change.
- JWT validation occurs before any route matching — an unauthenticated request is rejected at the gateway and never reaches a downstream service.
- Route configuration in `appsettings.Production.json` uses Docker Compose service names (e.g., `http://job-service:8080`) as cluster addresses, which only resolve inside the Docker Compose bridge network.
