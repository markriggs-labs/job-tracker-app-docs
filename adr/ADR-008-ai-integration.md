# ADR-008: AI integration strategy — dual-model approach with server-side prompt assembly

**Status:** Accepted  
**Date:** 2026-05-09  
**Author:** Mark

## Context

The Job Tracker application needs to help users generate tailored, ATS-optimized resumes by combining three inputs: the stored job description, the user's work experience document, and a set of user-defined generation instructions (AI profile). A decision is required on how to integrate AI capabilities into the application given the constraints of a portfolio project where ongoing API costs must be controlled.

## Decision

Two parallel integration paths are implemented under a single AI tab on the Job Detail page:

**Path 1 — Claude.AI Model (available now):** The AI Service assembles a complete, ready-to-use prompt server-side by fetching the job description from the Job Service and the experience document content from the Experience Service. The assembled prompt is returned to the frontend, where the user copies it and pastes it into their own [claude.ai](https://claude.ai) subscription. No Anthropic API key is required in the application.

**Path 2 — API Service Model (coming soon):** Direct in-app generation using **Claude 3.5 Sonnet** via the Anthropic SDK (v4.0.0). Resumes are generated and stored in MinIO without leaving the application. This path is built and wired but disabled in the UI until an Anthropic API key is provisioned for production.

## Rationale

- **Cost control:** Running a public portfolio site with open Anthropic API access creates unbounded billing risk. The Claude.AI path allows full feature demonstration without that risk — users bring their own subscription.
- **Demonstration value:** Path 2 is fully implemented (service, storage, API endpoint) and visible in the UI as "Coming Soon." This demonstrates the architectural capability — AI profile management, token forwarding, MinIO storage, async generation — without incurring production costs.
- **Server-side prompt assembly:** Assembling the prompt on the server rather than the client ensures the job description and experience content are fetched and composed in one place. It also provides a clear audit point and makes the prompt portable — the same server endpoint serves both paths.
- **TXT-first experience content:** The Experience Service reads TXT files back as plain text and embeds them directly in the prompt. PDF and DOCX formats are noted in the prompt as requiring manual attachment, which is the natural workflow for Claude.ai.

## AI profile system

Users define free-text instruction profiles (AI profiles) that are injected into every prompt. The AI Profile Instructions field is the **sole surface for all prompt customization** — no generation rules are hardcoded in the service. This gives users full control over generation style, industry emphasis, and formatting preferences without requiring service changes. Profiles are stored per-user in PostgreSQL via the AI Service.

## Alternatives considered

- **Client-side prompt assembly:** Simpler, no API call needed. Rejected because the server has access to all relevant data sources and can provide a better, more complete prompt without requiring the frontend to fetch multiple services independently.
- **Single always-on API integration:** Correct long-term approach but introduces production billing risk for a portfolio project. Deferred, not rejected.
- **Third-party AI abstraction layer (e.g., LangChain):** Adds dependency complexity without sufficient benefit at this scale. Anthropic SDK used directly.

## Consequences

- `ANTHROPIC_API_KEY` is a required environment variable but can be left blank — the Claude.AI path works without it.
- Generated resumes from Path 2 are stored in MinIO under `generated-resumes/{userId}/{jobId}/{resumeId}/` and listed on the AI tab.
- The AI Service forwards the user's Bearer token to the Job Service and Experience Service to enforce per-user data isolation.
- The AI Profile is the only place to configure prompt behavior. Hardcoded rules that were present in early iterations (experience depth, date ranges, embellishment guardrails) were removed — all such constraints now live in the user-managed AI Profile, making the generation behavior fully configurable without code changes.
- Future work: enable Path 2 in production by provisioning an API key; add PDF/DOCX text extraction (e.g., PdfPig, DocumentFormat.OpenXml) to eliminate the manual attachment step for binary experience documents.
