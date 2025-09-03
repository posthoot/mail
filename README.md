# Posthoot — Open‑source Email Marketing Engine

[![Docs](https://img.shields.io/badge/docs-docs.posthoot.com-0ea5e9?style=for-the-badge)](https://docs.posthoot.com)

<div align="center">
 <img alt="image" src="https://framerusercontent.com/images/REPwDVgxt71ZwZbIiQTntxA9h8Y.png" />
  <p><em>First-class email automation for modern enterprises</em></p>
</div>

Posthoot is an open-source, developer-first email marketing engine that gives you full control over your email infrastructure: connect multiple SMTP providers, manage campaigns and templates, bring AI into the workflow, and run marketing on autopilot.

## Vision

We built Posthoot to put teams back in control of their email stack.

- No vendor lock‑in; switch providers without replatforming
- Transparent, customizable, and self-hostable
- Real AI features to optimize content, timing, and segmentation
- Campaign automation that can run itself while you sleep

## Why Posthoot

- Multi‑provider SMTP routing for cost, reliability, and flexibility
- First‑class template control (HTML/CSS, dynamic content, A/B testing)
- Campaign automation (multi‑step sequences, triggers, goals)
- Rich analytics (opens, clicks, heatmaps, trends, audience insights)
- Enterprise‑ready: RBAC, API keys, rate limiting, auditability

## Architecture

- Server: Go (REST API, Swagger/OpenAPI), PostgreSQL, Redis
- Client: Next.js (App Router), TypeScript, Tailwind/shadcn, NextAuth
- Infra: Docker, optional K8s, Nginx, object storage

```
repo/
  server/   # Go API, OpenAPI, services, middleware
  client/   # Next.js web app
  docs/     # Documentation site (API Reference, Guides, Knowledge Base)
```

## Quick Start

- Self-hosting (server): see Guides → Self‑Hosting
  - Docker, Systemd, or Kubernetes deployment
- Client (Next.js UI): see Client → Setup
- OpenAPI/SDKs: see API Reference tab

Links:
- Docs home: https://docs.posthoot.com
- Self-hosting: https://docs.posthoot.com/guides/self-hosting
- Client setup: https://docs.posthoot.com/client/setup
- Rate limiting: https://docs.posthoot.com/rate-limiting

## API Reference

- Generated OpenAPI 3.0 (paths grouped under the API Reference tab)
- File uploads via multipart/form‑data requestBody
- Consistent error shape; rate‑limit headers on all responses

If your hosted OpenAPI isn’t publicly fetchable by your docs host, point docs to a local `./server/openapi.json` (ensuring valid OAS3 and permissive CORS when hosted).

## Knowledge Base (Literature)

Curated, practical guidance for operating email at scale:

- Troubleshooting: https://docs.posthoot.com/knowledge-base/troubleshooting/common-errors
- Deliverability best practices: https://docs.posthoot.com/knowledge-base/best-practices/email-deliverability
- Rate‑limit optimization: https://docs.posthoot.com/knowledge-base/performance/rate-limit-optimization

## Client (Next.js)

- Overview: https://docs.posthoot.com/client/overview
- Environment: https://docs.posthoot.com/client/env
- Development: https://docs.posthoot.com/client/development
- Testing: https://docs.posthoot.com/client/testing
- Deployment: https://docs.posthoot.com/client/deployment

## Server (Go)

- Middleware: rate limiting, auth, observability
- OpenAPI generation: `server/scripts/generate-openapi.sh`
- Swagger2 → OpenAPI3 conversion with body/formData normalization

## Contributing

- Fork, branch, PR (conventional commits appreciated)
- Write tests where it matters
- Keep code small, typed, and composable

## License

MIT — see `LICENSE`.

---

If you’re new here, start with:
1) Vision (https://docs.posthoot.com/vision) → 2) Introduction (https://docs.posthoot.com/introduction) → 3) Quickstart (https://docs.posthoot.com/guides/quickstart) → 4) API Reference (see the API Reference tab in docs).
