# Xem analytics implementation

Implemented September 10, 2026 from the analytics product review. Analytics changes are committed and pushed to `sudo`: backend [8721a0f](https://github.com/mailxem/xem.go/commit/8721a0f), frontend [7d33ed1](https://github.com/mailxem/xem-app.ts/commit/7d33ed1). Deployment verification is separate.

## Published checkpoint

Before analytics changes, the previous workspace work was committed and pushed to `sudo` in both repositories:

- Backend: [923917b](https://github.com/mailxem/xem.go/commit/923917b)
- Frontend: [ce33034](https://github.com/mailxem/xem-app.ts/commit/ce33034)

GitHub redirected the old `posthoot` repository URLs to the `mailxem` organization.

## Implemented

- Shared Analytics navigation: Overview, Audience, Campaigns & newsletters, Delivery health. Existing audience/team/trend URLs route into the shared reporting experience. Audience analytics is no longer under Settings & workspace.
- Date ranges, timezone, list/tag filters, previous-period comparisons, freshness, explicit rate denominators, retry/error states, and empty states. Date inputs show an inclusive end date; the API uses half-open ranges.
- Deduplicated active subscribers, new contact records, recipients reached, audience click rate, message click rate, directional open rate, captured bounce/complaint outcomes, failures, and uncertain delivery outcomes.
- Daily unique activity charts, five mutually exclusive engagement cohorts, and paginated cohort contacts with preserved report cutoff and filters. Contact links open the existing CRM search.
- Server-paginated and sortable list, campaign/edition, sending-domain, acquisition-source, and per-link breakdowns, with API totals. Page CSV export is explicitly labeled and escapes spreadsheet formula prefixes.
- Subscription-history baseline plus append-only status snapshots, written by PostgreSQL triggers. Captures imports, bulk/skip-hook writes, changes to contact identity/list/status, deletion, list archival/restoration, unsubscribe, and resubscribe. Repeated no-op writes create no new events; rolled-back writes roll back history too.
- Historical net change and a subscriber chart become available only for ranges entirely covered by recorded history. No fabricated backfill from contact creation dates. Historical tag filtering remains unavailable because tag membership history is not recorded.
- Signed message attribution on new newsletter unsubscribe links. Legacy resubscription retains old unsubscribe events, requires a signed token, and confirms through POST. Link scanners cannot opt a recipient in or out merely by fetching the confirmation page.
- Regular campaign links now carry the email ID instead of the campaign ID. New newsletter messages include tracking. Link rewriting preserves styling attributes, decodes HTML URL entities, and excludes unsubscribe/mailto links from click tracking.
- Analytics resource permissions and tenant/resource ownership apply to the entire analytics route group, including legacy exports. The Next proxy derives workspace identity from the session, validates resources, forwards filters, disables caching, and cancels/limits requests.
- Legacy detail rates now divide by accepted messages from the same send cohort; per-link unique counts are per destination, and missing ContactID no longer collapses all messages into one user. Legacy trends use send cohorts. Unsubstantiated optimal-send-time results are no longer returned.
- Removed the old demographic/preferences panels and components that plotted contact UUIDs as segments.

## API

All routes are below `/api/v1/analytics`, require authentication, and require analytics read access. Workspace IDs are derived from verified context; conflicting `teamId` parameters fail.

| Route | Response |
| --- | --- |
| `GET /report` | `metricVersion: audience-v2`, summary/previous period, daily activity, cohorts, growth coverage, definitions |
| `GET /audience` | Alias of the new report |
| `GET /breakdown` | `items`, `totalCount`, `page`, `pageSize`, `asOf` |
| `GET /people` | Unique active recipient page for the selected cohort |
| `GET /options` | Workspace list/tag choices (bounded to 200 per collection) |

Shared filters: `from`, `to` (exclusive), IANA `timezone`, optional `listId`, `tagId`, `campaignId`, and `asOf`. Maximum range: 366 days. Breakdown supports `kind=lists|campaigns|domains|sources|links`, `page`, `limit` (maximum 100), validated `sort`, and `direction`. Cohort contacts also support `cohort` and literal `search`.

Rates are `{ value, numerator, denominator }`; `value` is a fraction or null when the denominator is zero. The client formats percentages once. Legacy detail endpoints retain percentage units and emit a deprecation header.

## Data semantics and limits

- Recipient identity is a trimmed, case-normalized email address scoped to a workspace. List memberships and consent remain separate. Active subscribers require at least one active selected membership and no active workspace suppression.
- Report messages are non-test campaign/edition emails accepted by SMTP. CC/BCC and unresolved multi-recipient address formats are excluded. This does not prove inbox placement or confirmed delivery.
- Clicks and opens are observed events, not verified human activity. No bot classifier or privacy-proxy confidence score is claimed.
- Summary rates use messages sent in the selected period and events observed by cutoff. Daily activity can include events from older messages. Previous periods close at their own cutoff; differing message ages are disclosed.
- Subscriber growth measures recorded ACTIVE membership states, not deliverability eligibility. Current active subscriber counts additionally honor the workspace suppression list. These populations are labeled separately.
- Acquisition reporting groups form/import/unknown provenance. It does not infer individual form identity, conversion attribution, or revenue. Older unattributed opt-outs remain represented by status history after its start, not assigned to an invented campaign.
- List/tag reporting applies current membership filters. Growth is disabled for tag filters. Campaign-specific past membership changes are not reconstructed as historical campaign audience membership.
- Date filters do not retroactively change the “active subscribers now” card. No historical current-size comparison is shown.
- Advanced preference models, revenue attribution, verified delivery, bot filtering, and statistically supported send-time optimization remain future work, as described in the review.

## Release requirements

Deploy the backend before enabling the updated frontend against a hosted API. The existing backend migration runner installs the nullable email tracking-capability column, analytics indexes, the history tables, baseline, and contact/list triggers in its migration transaction. The migration database role needs table/index/function/trigger DDL permissions. Initial indexing/baselining can lock tables; use the normal database backup and migration release procedure for a large production database.

The local frontend is configured to use the hosted API. Local Go changes alone do not update that API. Until the backend is deployed, the UI reports analytics unavailability rather than showing fabricated zeros.

The new `/audience` response replaces its old contact-keyed shape. Consumers of that response must move together with this release. Subscription history starts at migration time and cannot truthfully reconstruct missing past transitions.

## Validation

- Production Next build and TypeScript checks passed.
- Frontend: 21 tests passed, including null-vs-zero formatting, percentage-point comparison, anonymous/cross-workspace proxy rejection, resource allowlisting, and filter/error propagation.
- PostgreSQL integration tests passed for deduplication, repeated tracking, missing contact associations, date windows, pagination/totals, cohort reconciliation, tenant rejection, tag filters, suppression, status history, idempotency, rollback, and archive/delete tombstones.
- Relevant backend packages passed: analytics, handlers, marketing, utils, tasks, API, services, and automation.
- Browser checks passed on desktop and at 390px: one title per page, chart rendering, API pagination, cohort cutoff preservation, invalid dates, retry, and no document-level horizontal overflow. Browser data came from real PostgreSQL test queries on synthetic records, not the live workspace.
- Scale check: 10,000 contacts / 50,000 accepted messages, main report approximately 0.7 seconds on the local test database. All breakdowns completed; the full seed/report/breakdown fixture test took approximately 6.5 seconds. This is a local measurement, not a production latency guarantee. Query-plan inspection exposed poor estimates from COALESCE predicates and materialized contact expressions; those were corrected.
- The unrestricted `go test ./...` run is blocked by pre-existing `internal/ai/agent/knowledge_test.go` fixtures referring to removed fields (`Contact.Name`, `EmailTracking.TeamID/Status/OpenedAt/ClickedAt`, `EmailStatusDelivered`). That package was not changed to hide the failure.

Reproduce database integration tests with an isolated PostgreSQL database:

```sh
POSTHOOT_TEST_DATABASE_URL='host=localhost port=5432 user=TEST_USER dbname=TEST_DB sslmode=disable' go test ./internal/analytics -v
```

Tests create and remove their own schema. Set `ANALYTICS_SCALE_TEST=1` to include the representative dataset check. Never point schema-creating tests at a production database.

## September 10 chart extension (local implementation)

The real analytics dashboard now uses the website's multicolor shadcn/Recharts visual language: interactive campaign/newsletter stacks, subscriber growth area, engagement mix, daily activity, reported click devices, and time-of-day heatmap. Existing query filters, server pagination, cohort drilldowns, rate definitions, error states, and history-coverage behavior remain in use. These changes are not yet deployed to the hosted API.

`GET /report` adds `charts` within the same repeatable-read transaction:

- `volume`: daily `campaigns`, `newsletters`, `campaignClicks`, `newsletterClicks`. Volumes reconcile to `summary.accepted`; clicked-message series reconcile to `summary.clickedMessages`. These are send cohorts. Automation/transactional mail outside the existing marketing scope is not mislabeled as a third series.
- `devices`: desktop, mobile, tablet, other, unknown counts derived from recorded `device_type`.
- `clickHours`: 168 day/hour buckets, Monday=0, using the selected IANA timezone.
- `clickEvents`: observed event total shared by device and hour buckets. Repeated clicks and clicks on older accepted messages are included; deleted events, events before send time, foreign workspace emails, and events outside the selected window are excluded. These are not verified-human counts or send-time recommendations.

The client groups hours into accessible four-hour cells, provides device selection, a daily values table and CSV export, and respects reduced motion. It shows an explicit unavailable message if an older API omits the new aggregate fields. Historical subscriber growth still requires complete recorded history; it is never filled with synthetic data.

New PostgreSQL coverage checks stack/summary reconciliation, device/hour reconciliation, timezone boundaries, current list/campaign filters, repeat events, older-message clicks, corrupt cross-workspace associations, and empty intersections. The representative 10,000-contact/50,000-message report completed in approximately 0.8 seconds locally including the new queries. No schema migration is required for this extension; the previous analytics migration remains a prerequisite.

## Interactive chart release — September 10, 2026

Published API chart aggregates in backend `61fba87` and interactive dashboard charts in frontend `eb7123d`. Campaign/newsletter volume and click bars reconcile to report totals; device and weekday/hour charts count observed click events with workspace and report filters. PostgreSQL analytics integration tests and frontend TypeScript checks passed before publication. Deployment remains a separate operation.

## Dashboard Email / Campaigns tabs — September 10, 2026

The homepage now defaults to Email, with a separate Campaigns tab reusing the marketing analytics workspace. The single Dashboard heading includes Outbox and Create campaign actions. Empty periods retain metrics and charts with a useful next step.

`GET /api/v1/analytics/email-overview` returns `metricVersion: email-v1`, `summary`, rates with explicit numerators/denominators, and daily `series`. It uses the same analytics authentication and workspace permissions as other reports. The Next proxy accepts `email-overview` and scopes it to the signed-in workspace.

This report includes all nondeleted, nontest email records, without requiring a campaign, contact, or list. Accepted sends are grouped by send timestamp; unsent/unknown records by creation timestamp. Pending, failed, and draft counts represent current statuses of records created during the range. A legacy SENT record without a valid send timestamp is unknown. Counts are email records, not individual CC/BCC recipients. Known bounces overlap sent counts. Engagement counts deduplicate per message and use events after sending and before the report cutoff. General email reports accept date/timezone filters; audience filters belong in Campaigns.

Published backend `374769d` and frontend `611af22`. Validation: PostgreSQL analytics/handler/API tests, six frontend analytics tests, TypeScript, production build, and Chrome checks covering both tabs, one heading, standalone email activity, empty states, inclusive dates, errors/retry, and mobile overflow. Temporary test servers and the database container were stopped afterward. Deploy the backend before the frontend so the new endpoint is available; a missing endpoint is shown as an error, not fake zero activity.
