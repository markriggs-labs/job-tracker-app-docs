# ADR-003: Auth0 for identity and access management

**Status:** Accepted  
**Date:** 2026-04-11  
**Author:** Mark

## Context

The application is multi-user and requires secure authentication, user registration, password management, and session handling. A decision is required on whether to build a custom auth system or use a managed identity provider.

## Decision

Auth0 will be used as the managed identity provider. It handles login, registration, password reset, email verification, and JWT token issuance. The application trusts the JWT token issued by Auth0 and uses the user ID claim to scope all data access.

## Rationale

- Auth0 eliminates the need to build and maintain authentication infrastructure, which is a significant security risk if implemented incorrectly.
- The free tier supports up to 25,000 monthly active users, which is more than sufficient for a portfolio application.
- Auth0 provides a hosted login page, registration flow, and password reset out of the box with no custom UI required for these flows.
- JWT tokens are industry standard and integrate cleanly with API gateways and individual microservices for token validation.
- Auth0 is owned by Okta and is widely used in enterprise applications, making it a credible and recognizable portfolio choice.
- Using a managed identity provider reflects real-world engineering judgment — building custom auth is rarely the right decision.

## Alternatives considered

- **Custom JWT implementation:** Higher risk, higher maintenance, not advisable for a production system.
- **Okta Workforce Identity:** Better suited for employee identity management rather than customer-facing applications.
- **Firebase Authentication:** A viable alternative, but Auth0 has stronger enterprise recognition and more flexibility.

## Consequences

- Application is dependent on Auth0 availability, though this is mitigated by Auth0's 99.99% uptime SLA.
- No custom user management admin panel needs to be built — Auth0's dashboard handles this.
- All user data and credentials are managed by Auth0, reducing compliance burden on the application.
- Future MFA support can be enabled through Auth0 configuration without code changes.
- A Post-Login Auth0 Action injects the user's email as a custom claim (`https://job-tracker/email`) into the access token so downstream services can identify the user without calling back to Auth0.
- The `offline_access` scope is requested during login to instruct Auth0 to issue a refresh token alongside the short-lived access token. Without this scope, Auth0 issues access tokens only and the session cannot be silently renewed after expiry.
- Access tokens are cached in localStorage (via the Auth0 React SDK's `cacheLocation: 'localstorage'` option) so the session survives a full page refresh. This is a deliberate trade-off: localStorage is accessible to JavaScript within the same origin, which is an accepted risk for a self-hosted, single-user portfolio application where the alternative (losing the session on every refresh) is a worse user experience.
- The React frontend uses an axios request interceptor to handle 401 responses. When a 401 is received, the interceptor silently calls `getAccessTokenSilently()` to obtain a fresh token and retries the original request. If the refresh fails (expired or revoked refresh token), the interceptor calls `logout()` to return the user to the Auth0 login page. This prevents silent data access failures after session expiry without requiring the user to manually sign in.
