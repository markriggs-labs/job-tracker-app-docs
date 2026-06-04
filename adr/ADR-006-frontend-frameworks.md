# ADR-006: React as the initial frontend framework, with Angular as a planned second implementation

**Status:** Accepted  
**Date:** 2026-04-11  
**Author:** Mark

## Context

The application requires a dynamic single-page application with complex UI state — multi-step forms, status boards, activity journals, and file upload flows. A decision is required on the frontend framework. The architecture should also support a second frontend implementation to demonstrate framework-agnostic API design.

## Decision

React will be used as the initial frontend framework. An Angular implementation will follow as a second frontend, consuming the same API gateway. Both frontends will be built as SPAs served as static builds, with all API communication handled via HTTPS calls to the API gateway. The API layer will remain completely frontend-agnostic.

## Rationale

- React is the most widely used frontend framework in enterprise environments, making it the most recognizable starting point for a portfolio application.
- Building a second Angular frontend against the same API directly demonstrates that the backend was designed correctly — frontend-agnostic, contract-driven, and decoupled from any presentation layer.
- The dual-frontend approach is a strong portfolio differentiator. It shows breadth across both major enterprise frameworks and reinforces the architectural principle of separation of concerns.
- Auth0 provides well-supported SDKs for both React and Angular, enabling consistent authentication implementation across both frontends.
- React and Angular are the two most commonly required frameworks in enterprise engineering job descriptions, making both implementations directly relevant to the job search.
- Starting with React allows faster initial progress while the Angular implementation follows as a Phase 2 effort.

## Alternatives considered

- **Vue.js:** Lightweight and approachable but less prevalent in enterprise environments. Not selected as it would not add the same portfolio value as Angular.
- **Server-side rendering (Next.js):** Adds complexity not justified for an authenticated single-user application.
- **Angular only:** Would miss the opportunity to demonstrate React experience, which has broader market coverage.

## Consequences

- The API gateway and all microservices must expose clean REST contracts with no assumptions about the consuming frontend.
- Both frontends will need separate build pipelines configured in GitHub Actions.
- Environment-specific configuration (API URLs, Auth0 tenant) is managed via environment variables at build time for both implementations.
- The Angular implementation will be tracked as a Phase 2 deliverable once the React frontend reaches feature parity.
- Maintaining two frontends increases long-term maintenance surface but the portfolio and learning value justifies this for the current use case.
- **All API calls from the React frontend — including the SignalR WebSocket connection — are routed through the API Gateway via a single `VITE_GATEWAY_URL` environment variable.** The frontend has no direct knowledge of individual service URLs, keeping the internal service topology completely hidden from the browser.
- **TanStack Query (React Query)** is used for all server state management. It provides request deduplication, background refetching, and targeted cache invalidation. SignalR `jobCreated` and `jobUpdated` events trigger query invalidations rather than full page refreshes, keeping the UI in sync with the database after each Kafka-driven write.
- **The React frontend is deployed to GitHub Pages** as a static Vite build. A GitHub Actions workflow runs Vitest tests, builds the SPA, and publishes to the `gh-pages` branch on every push to `main`. The deployment is blocked if tests fail.
- **GitHub Pages does not natively support client-side routing** — a direct request to a deep link such as `/jobs/123` returns a 404. The standard redirect workaround is implemented: a custom `404.html` captures the full URL and redirects to `/?/jobs/123`, and `index.html` restores the original path via `history.replaceState`. This makes deep links and page refreshes work correctly without server-side configuration.
- **Vite's `base` URL is set conditionally at build time** using a `GITHUB_PAGES` environment variable. The GitHub Actions workflow sets this flag, causing the Vite config to use the repository path as the base (`/job-tracker-app-web-react/`). Local and VPS builds use the default base (`/`).
