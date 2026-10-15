# CivicSignal

CivicSignal is an evidence-first civic issue intelligence dashboard. It turns individual community reports into a searchable incident picture, flags possible duplicate/related complaints, makes uncertain matches explainable, and keeps a human reviewer in control of incident merges and claim verification.

> **Demo data notice:** the included Chicago sample reports are fictional and are not live 311 data. The sample OpenStreetMap links provide location context only, not proof of an incident.

## What is implemented

- Complaint submission with title, narrative, civic category, location, date/time, optional coordinates, evidence URL, source, and severity; input is validated server-side.
- Local explainable matching using TF-IDF-weighted cosine similarity plus location, category, and time signals. Possible pairs are labeled **Duplicate**, **Related**, **Uncertain**, or **Distinct** with score components and reasons. No paid API, model key, or external inference service is required.
- Each complaint remains an original `reports` row. High-confidence duplicate clusters can be attached automatically; other candidates enter the human review queue.
- Human review approve/reject, incident split, reviewer attribution, timestamped audit history, and undo for merge/reject/split actions.
- Incident management: status, severity, location, report count, evidence links, source reports, claims, and an event timeline.
- Evidence-backed claim tracking. Supporting evidence alone is not represented as confirmed; a human can verify only when supporting evidence exists and no contradictory evidence is recorded. Contradictions move the claim to **Contested**.
- Interactive Leaflet/OpenStreetMap report map with category, status, and severity filters. Marker size corresponds to severity; color corresponds to category.
- Search and filters for incident text/location, category, status, severity, and date range.
- Live dashboard metrics, activity chart, category breakdown, resolution metrics, and a labeled matching evaluation endpoint with duplicate precision, recall, F1, accuracy, and a confusion matrix.
- Fictional, repeatable seed data and a safe reset action that touches only records marked as demo data.
- JSON API error responses and a documented `/api/health` route.

## Architecture

This workspace is implemented in the supported **Next.js App Router + TypeScript + PostgreSQL + Drizzle ORM** stack. The UI is a responsive React client application; Next.js route handlers provide the backend API; Drizzle manages relational tables and transactions. This keeps the app deployable as one service in the provided environment. It does not use the originally suggested FastAPI/SQLite/Vite split stack.

- `src/app/` — dashboard, interactive components, and route handlers (`/api/*`).
- `src/db/schema.ts` — incidents, reports, matches, claims, claim evidence, and review action tables.
- `src/lib/matching.ts` — transparent TF-IDF/spatial/time matching engine.
- `src/lib/seed.ts` — advisory-lock protected demo seed and demo-only reset.
- `src/lib/evaluation.ts` — labeled illustrative benchmark and metrics.
- `tests/matching.test.ts` — executable matching and validation tests.
- `docker-compose.yml` — optional local PostgreSQL 16 service.

## Run locally (Windows PowerShell)

Requirements: Node.js 20.9+ (Node 22 recommended), npm, and either Docker Desktop or a local PostgreSQL server.

```powershell
# From the CivicSignal project folder
Copy-Item .env.example .env

docker compose up -d postgres
npm install
npx drizzle-kit push
npm run dev
```

Open **http://localhost:3000**. The demo data is seeded automatically on the first dashboard/API request. The PostgreSQL container uses trust authentication only for a loopback-bound local development port; do not expose this database outside your own machine. To stop the local database:

```powershell
docker compose down
```

To run the automated checks after applying the schema and starting PostgreSQL:

```powershell
npm exec tsx -- --test tests/matching.test.ts tests/contradictions.test.ts
npm exec tsx -- --test tests/api.integration.test.ts
```

The API integration test loads `.env`, uses the configured PostgreSQL database, and removes the temporary integration rows when it finishes.

To reload only the illustrative sample rows while preserving non-demo reports, first start the app and then use either the **Reset demo data** button in the sidebar or:

```powershell
Invoke-RestMethod -Method Post -Uri http://localhost:3000/api/seed -ContentType 'application/json' -Body '{"resetDemo":true}'
```

The seed/reset endpoint is intentionally a local-demo convenience, not an authenticated production admin endpoint. Disable or protect it before exposing the app publicly.

## API

All request/response bodies use JSON unless otherwise noted. Errors have a stable shape:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Please correct the complaint fields.",
    "details": { "title": "Use a clear title between 8 and 120 characters." }
  }
}
```

| Method | Endpoint | Purpose |
| --- | --- | --- |
| `GET` | `/api/health` | Liveness and PostgreSQL connectivity. |
| `GET` | `/api/dashboard` | Metrics, recent incidents, categories, and 7-day activity. Seeds the demo on first request if needed. |
| `GET` | `/api/incidents?search=&category=&status=&severity=&from=&to=` | Search/filter incident clusters. `status` accepts `all`, `active`, `open`, `investigating`, `verifying`, or `resolved`. |
| `GET` | `/api/incidents/{id}` | Incident details, original reports, claims/evidence, and timeline. |
| `PATCH` | `/api/incidents/{id}` | Update `status` and/or `severity`; change is audit logged. |
| `POST` | `/api/incidents/{id}/split` | Body `{ "reportId": "<uuid>" }`; split one original report into a new cluster. |
| `GET` | `/api/reports?search=&category=&from=&to=` | Search/filter preserved original reports. |
| `POST` | `/api/reports` | Submit a complaint; validates inputs, scores candidates, and returns the match/auto-cluster decision. |
| `GET` | `/api/map?category=&status=&severity=` | Geolocated report points and incident attributes for Leaflet. |
| `GET` | `/api/reviews?status=pending\|reviewed\|all` | Candidate comparisons plus traceable review history. |
| `PATCH` | `/api/reviews/{id}` | Body `{ "action": "approve" }` or `{ "action": "reject", "reviewer": "..." }`. Approve merges incident clusters; rejecting keeps them separate. `Distinct` cannot be approved. |
| `PATCH` | `/api/review-actions/{id}` | Body `{ "action": "undo" }`; restores assignments for a supported latest merge/reject/split action. |
| `GET` | `/api/claims?incidentId=&status=` | Claims with linked incident and evidence. |
| `POST` | `/api/claims` | Body `{ "incidentId": "<uuid>", "statement": "..." }`; optional `evidence` contains `kind`, `excerpt`, `source`, optional `url` and `reportId`. New claims are never born verified. |
| `PATCH` | `/api/claims/{id}` | Body `{ "action": "verify" }` or `{ "action": "unverify" }`. Verify requires support and no contradictory evidence. |
| `POST` | `/api/claims/{id}/evidence` | Add `supporting` or `contradicting` evidence. Contradictions automatically contest the claim. |
| `GET` | `/api/evaluate` | Executes the local classifier against 12 labeled illustrative pairs and computes confusion matrix, accuracy, duplicate precision/recall/F1. |
| `POST` | `/api/seed` | Body `{ "resetDemo": true }` safely reloads only `isDemo` rows; `{}` idempotently seeds if sample rows are absent. |

Important validation responses include `422 VALIDATION_ERROR` (field details), `409 REVIEW_ALREADY_DECIDED`, `409 SUPPORTING_EVIDENCE_REQUIRED`, `409 CONFLICTING_EVIDENCE`, `409 CANNOT_SPLIT_SINGLE_REPORT`, `404 INCIDENT_NOT_FOUND`, and `503` when the database is unavailable.

## Matching and evaluation notes

The local matcher normalizes tokens, computes weighted cosine similarity for titles/descriptions, then combines text (55%), location (23%), category (12%), and time (10%). Coordinates use a Haversine distance band; without coordinates, location wording is compared. Decision thresholds are explicit in `src/lib/matching.ts` and each API suggestion includes its component values and reasons. This is a lightweight, deterministic prototype—not a claim of state-of-the-art semantic AI accuracy. The hand-labeled synthetic evaluation exists to make behavior measurable and regression-testable, not to imply production-grade quality. Human review remains the final authority.

## Data safety and production hardening

The app has no authentication, authorization, or role management. It is a hackathon demo and should run on a trusted network until reviewer identity, rate limiting, moderation, privacy policy, and production database credentials are configured. Do not submit private or sensitive personal data. User-provided URLs are stored as links and never fetched server-side. The demo reset deletes only rows explicitly marked `is_demo = true`. Map basemap tiles are served by OpenStreetMap and require a browser network connection.
