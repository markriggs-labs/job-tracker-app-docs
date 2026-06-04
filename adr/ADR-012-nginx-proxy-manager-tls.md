# ADR-012: Nginx Proxy Manager for TLS termination

**Status:** Accepted  
**Date:** 2026-04-20  
**Author:** Mark

## Context

The application is deployed on an Akamai VPS with all backend services running inside a Docker Compose network. External HTTPS traffic from the browser must be terminated at the VPS boundary before reaching the API Gateway. A decision is required on how TLS certificates are managed and how inbound HTTPS requests are forwarded to the internal Docker network.

## Decision

**Nginx Proxy Manager (NPM)** is used as the TLS termination layer. NPM runs in a separate Docker Compose stack (`nginx-proxy-manager/docker-compose.yml`) on the same VPS, connected to the application stack via a shared Docker bridge network (`proxy`). It handles:

- Automatic Let's Encrypt certificate provisioning and renewal for the production domain (`jobtracker-dev.duckdns.org`)
- HTTPS termination — decrypts inbound traffic on port 443 and forwards plain HTTP to the YARP API Gateway on the internal Docker network
- HTTP-to-HTTPS redirect on port 80

DuckDNS provides dynamic DNS, mapping the production subdomain to the VPS public IP. A `duckdns-updater` container in the application stack periodically refreshes the DNS record.

## Rationale

- **Admin UI:** NPM provides a web-based admin panel for managing proxy hosts and certificates, making it faster to configure and troubleshoot than writing nginx config files directly.
- **Automatic Let's Encrypt:** NPM handles certificate provisioning and renewal through its UI — no manual certbot commands or cron jobs required.
- **Separate stack:** Running NPM in its own `docker-compose.yml` isolates the TLS layer from the application stack. The application stack can be restarted or redeployed without dropping the TLS proxy.
- **Shared Docker network:** The `proxy` bridge network allows NPM to reach the API Gateway by Docker service name without exposing any internal port to the public host network.
- **DuckDNS compatibility:** Let's Encrypt HTTP-01 challenge works with DuckDNS-resolved domains — NPM handles the challenge automatically on port 80.

## Alternatives considered

- **certbot + nginx directly:** More control, no admin UI dependency. Requires manual nginx config editing and a cron job or systemd timer for cert renewal. Higher operational overhead for a single-developer setup.
- **Caddy:** Automatic HTTPS with a simple `Caddyfile`. Strong alternative but less familiar and less commonly discussed in enterprise contexts than nginx.
- **Traefik:** Docker-native reverse proxy with automatic service discovery via container labels. Powerful but adds complexity — container label configuration is less transparent than NPM's UI for simple single-service routing.
- **Cloudflare Tunnel:** Zero-trust proxy that eliminates the need for open inbound ports. Viable but introduces a third-party dependency in the critical request path.

## Consequences

- NPM introduces a double-hop proxy chain: **Browser → NPM (TLS termination) → YARP API Gateway → Kestrel**. This chain prevented reliable WebSocket frame forwarding through both proxies in sequence, which drove the SignalR LongPolling decision documented in ADR-007.
- NPM's default `proxy_read_timeout` is 60 seconds. SignalR Long Polling's default poll timeout (90 seconds) exceeded this, causing 504 Gateway Timeout errors. The resolution — `proxy_read_timeout 300s` in NPM's custom nginx config, and `LongPolling.PollTimeout = 45s` in the SignalR hub — is documented in ADR-007.
- Certificates are scoped to the DuckDNS subdomain. If the production domain changes, certificates must be re-provisioned through the NPM UI.
- The NPM admin panel runs on port 81. It is not exposed to the public internet in the current Terraform firewall configuration (see ADR-009), which allows only ports 22, 80, and 443 inbound.
- A single-hop alternative — exposing Kestrel directly behind NPM and bypassing YARP — would resolve the WebSocket issue but would remove the API gateway architectural layer. This trade-off is discussed in ADR-007 and ADR-011.
