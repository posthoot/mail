# Xem workspace release notes

Updated September 10, 2026. Source changes are published to GitHub; no production deployment or real email send was performed.

## Application changes

- The dashboard and authentication screens use the Xem branding and shared Tailwind/shadcn styling. The viewport is bounded, with the dashboard content scrolling below its header. Outbox remains at `/developer/logs/emails` and shares the Inbox interface.
- The top bar has workspace search on the left and Template library/account controls on the right. Repeated page labels and the sidebar account menu are removed; each page keeps its main heading. API keys and Webhooks are grouped under Settings & workspace and remain available through search.
- Templates, mailing lists, contacts, campaigns, API keys, SMTP, IMAP, tags, webhooks, and settings use the shared collection cards where appropriate. Activity logs retain tabular presentation.
- Charts reuse `components/ui/chart.tsx`, including shadcn tooltips and legends. Engagement and trend comparisons use grouped bars. Date filters reach the backend, and rates are converted to percentages once.
- Resource providers render React Query data directly. Cached navigation no longer relies on query-function side effects. Pagination uses the API's one-based page numbers; total counts are calculated before applying pagination.
- Single-record responses support raw records and older `{data: record}` adapters. The reported list-detail undefined-query error is covered by regression tests. The user's All Users list loaded with 36 contacts during browser verification earlier in this task.
- The existing Unlayer editor is retained. Cached templates hydrate once per record, background refetches do not erase edits, and export/save failures preserve the draft. SMTP/IMAP dialogs clear edited values before adding a new connection.

## Features and reuse

Newsletters reference existing templates, contact lists, and senders. Four starter templates import editable Unlayer designs. Scheduled editions snapshot template content into existing campaigns; the scheduler supports daily, weekly, and monthly recurrence with timezone handling and a monthly anchor day. Delivery keys prevent duplicate task materialization. Suppression and unsubscribe behavior is preserved.

The React Flow automation builder persists the existing graph model and exposes validation, publish/pause, and execution history. Lead forms reuse existing form/contact models, include consent capture and retry receipts, and can be published and embedded. CRM uses existing contacts with stages and notes. Pagination, name/email/company search, and stage filters now run in the API without the former 200-contact cap. The API returns the filtered total and workspace-wide summary counts; both the table and pipeline have pagination, with pipeline column counts explicitly limited to the current page.

The email writer uses the supplied OpenAI-compatible proxy from the Go backend. It generates plain-text subjects and bodies for review in Compose or the existing template editor. It cannot send mail. Applying a draft in the template editor explicitly replaces its copy/design; existing template save remains in use.

Tags use existing `Tag` and `Contact.Tags` associations, with workspace-scoped management and assignment. The tags UI supports creation, rename, deletion, contact counts, and contact assignment.

Profile edits persist the authenticated user's name/bio. Branding persists the existing TeamSettings/BrandingSettings records and workspace name; logo input uses an HTTPS image URL. Domain ownership is verified with a unique `_xem.<domain>` TXT record. This does not provision sending-provider DKIM/SPF, custom-domain TLS, or domain routing.

Webhook status and delivery history have scoped backend handlers. API-key usage is scoped through its parent key because usage rows have no TeamID column; the former generic usage routes now use this handler. Usage metrics and exports explicitly describe the selected page, with no invented latency statistics.

Invitation acceptance uses the existing endpoint and now creates the user, permissions, and invitation status in one transaction. A failure to create permissions rolls back user creation.

## Backend routes

Existing authentication and permissions still apply. New marketing routes are under `/api/v1/marketing`:

- `GET /contacts?page=1&limit=10&search=...&stage=QUALIFIED`: requires `contacts:read`; returns `{data,total,page,limit,summary}`. `total` counts matching, nondeleted workspace contacts; `summary` contains all workspace totals (`total`, `qualified`, `customers`, `subscribed`). Maximum limit is 100, and out-of-range pages clamp to the last page. Rows include a tenant-scoped `listName`. Counts and rows share a read-only repeatable-read transaction.
- `POST /email-draft`
- `GET/POST /tags`, `PUT/DELETE /tags/:id`, `GET/PUT /contacts/:id/tags`
- `PUT /profile`, `GET/PUT /branding`
- `POST /domains`, `POST /domains/:id/verify`
- `PUT /webhooks/:id/status`, `GET /webhooks/:id/deliveries`
- Newsletter, form, CRM note/stage, starter-import, and template-preview routes in `internal/routes/marketing_routes.go`.

Existing `/api/v1/api-key-usage` and `/api/v1/api-key-usage/:id` now enforce workspace ownership. Public form/unsubscribe routes are under `/public`.

## Schema and rollout

Back up the database and rehearse migration against a PostgreSQL staging copy before release. Startup AutoMigrate includes the newsletter/form receipt/contact note models, scheduling/delivery fields, User.Bio, and Tag.TeamID.

`models.BackfillTagWorkspaces` runs within the schema migration transaction. Each legacy tag with a NULL TeamID is copied into the workspaces identified by its contact associations. Contacts are reassociated with their own workspace's tag; the unassigned original is retained for audit. Shared tags therefore become independent per-workspace records. Tags with no contact associations remain unassigned because there is no evidence of ownership. Repeated migration does not create further copies. Stop old API/worker versions during this migration so they cannot continue creating unscoped tags.

Roll out the backend, scheduler/worker, and frontend together. Verify the new routes before exposing their controls. The current local client configuration points at **api.xem.email**, so changing the local Go source alone does not make the new routes available in the authenticated UI. Nothing was deployed as part of this work.

Environment:

| Variable | Meaning |
| --- | --- |
| `AI_PROXY_BASE_URL` | Defaults to `https://ai-proxy.synehq.com/v1` |
| `AI_PROXY_MODEL` | Defaults to `gpt-4.1` |
| `AI_PROXY_API_KEY` | Optional; the supplied proxy is open and does not require a secret |
| `PUBLIC_API_URL` | Externally reachable API base used for public/unsubscribe URLs |
| `CORS_ALLOWED_ORIGINS` | Comma-separated approved frontend origins; development default is `http://localhost:3000` |
| Existing auth, database, Redis, storage, SMTP settings | Retain the established configuration and secret management |

AI requests use an HTTPS endpoint, disallow redirects, bound request/response sizes, and time out. The per-workspace AI limit and public-route limits use process memory; a multi-replica release needs an edge/distributed limit. Infrastructure manifest changes require review against the target cluster, especially network-policy egress for DNS, database/Redis, SMTP/IMAP, storage, and the AI proxy. They have not been applied or externally audited.

## Validation and remaining limits

Passed during this task:

- Next production build, TypeScript, and all eight frontend regression tests.
- Go build and tests for AI, marketing, automation/processors, handlers, services, and API packages.
- Regression coverage for cached navigation, response shapes, pagination counts, tag lifecycle/isolation, legacy-tag splitting/idempotency, profile/branding/API-key log ownership, domain input/ownership, and invitation rollback.
- PostgreSQL integration tests for the marketing package, including CRM pagination beyond 200 rows, filtered totals, summary counts, literal search, stable ordering, page clamping, and workspace isolation. The Docker daemon became available for this rerun. Earlier newsletter/form/automation PostgreSQL tests also passed; workspace/tag-migration checks outside this package still need a PostgreSQL rerun.
- A live AI-proxy smoke request with fictional text returned a valid GPT-4.1 subject/body. No contact content or email sends were used.

The header layout, account dropdown, nested Settings navigation, and CRM filtered totals were visually verified in the local preview after this update. CRM API behavior was verified through handler integration tests; the authenticated client still requires the backend route rollout. The CSS repair and reported contact list were verified in Chrome earlier. Final visual verification of every card page, auth state, mobile width, and AI interaction remains incomplete because the Mac was locked. Run those checks after unlocking, including refresh, client navigation, and retry for the reported list URL.

The full repository Go test suite has pre-existing compilation failures in `internal/ai/agent/knowledge_test.go` from obsolete model fields. The targeted package results must not be represented as a passing full suite.

Notification preferences do not have an existing delivery implementation. The settings page now says they are unavailable instead of presenting switches that silently discard changes. This release does not claim full notification delivery, provider authentication setup, security certification, or production readiness without staging and deployment verification.

The task-created PostgreSQL container was named `posthoot-marketing-test`. The Docker daemon was unavailable at cleanup time; if that container returns when Docker starts, it can be removed. The unrelated service on local port 8080 was left alone. The Next development server remains on port 3000.

## Template library, AI design, and compose follow-up

- 218 static native Unlayer starters: 20 reference-inspired layouts, 18 originals, and 180 distinct editorial compositions. Removed the former industry reskins. Retained source references and generated contact sheets for review.
- Starter gallery loads in 12-item scroll batches, with category, search, and collection filters. It has no marketing-options API dependency.
- Starter designs load into the existing editor and save using Unlayer-exported HTML. Local preview images are adapted for Chrome's loopback restrictions and restored to hosted URLs during export. Deploy the public asset directory with the client.
- Template AI now requests `format: design`; the Go service validates a bounded layout specification and compiles safe native Unlayer JSON. Deploy the updated backend before enabling this against the hosted API. Outbox continues to request personal email copy and now reuses the original Maily rich editor with image support.
- Auth pages use the sidebar icon plus a dark Xem wordmark. Public app icons are excluded from authentication middleware.
- Checks cover registry integrity, distinct block structures, editor asset export, and AI output validation. Browser checks use isolated local QA sessions and intercepted writes; no email is delivered.
