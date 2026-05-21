# Threat Model

## Project Overview

Travel Next Level is a publicly deployed Replit app with two production services: a static landing page at `/` from `artifacts/travel` and a small Express API mounted at `/api` from `artifacts/api-server`. The current production API surface is minimal and exposes only a health check. The deployed frontend is primarily a static HTML page with inline JavaScript and third-party CDN-hosted libraries.

## Assets

- **Site integrity and visitor trust** — the landing page is public and brand-facing. If an attacker can alter client-side code, they can deface the site, redirect users, or execute arbitrary JavaScript in visitors' browsers.
- **Production API availability** — the `/api` service is publicly reachable and must remain available for health checks and future expansion.
- **Application secrets and infrastructure configuration** — `DATABASE_URL`, runtime environment variables, and any future API credentials must remain server-side only.
- **Database connectivity** — the database package is wired into the workspace and would become high impact immediately if future routes expose it unsafely, even though the current production route set does not query it.

## Trust Boundaries

- **Browser to static frontend** — visitors load HTML, images, and JavaScript from the public deployment. The browser is untrusted, and any JavaScript executed in this origin has full access to the page.
- **Browser to API** — public requests cross into the Express service at `/api`. All request data must be treated as attacker-controlled.
- **Frontend to third-party CDNs** — the landing page loads runtime JavaScript from external origins. Those resources execute with the same privileges as first-party code once loaded.
- **API to database** — the server can connect to PostgreSQL through `DATABASE_URL`. SQL or data exposure issues here would directly impact confidentiality and integrity if database-backed routes are added.
- **Production vs dev-only artifacts** — `artifacts/mockup-sandbox` and draft files under `attached_assets/` are treated as non-production unless a production route or artifact explicitly serves them.

## Scan Anchors

- **Production entry points:** `artifacts/travel/index.html`, `artifacts/api-server/src/index.ts`, `artifacts/api-server/src/app.ts`
- **Highest-risk production area today:** `artifacts/travel/index.html` because it contains inline logic and third-party script imports; `artifacts/api-server/src/**` is small and currently exposes only `/api/healthz`.
- **Public surfaces:** `/` static landing page, `GET /api/healthz`
- **Authenticated/admin surfaces:** none currently implemented in production
- **Usually dev-only / ignore unless proven reachable:** `artifacts/mockup-sandbox/**`, `attached_assets/**`, generated client helpers not imported by production entry points

## Threat Categories

### Tampering

The main tampering risk is browser-side code integrity. The public landing page executes multiple third-party JavaScript libraries loaded from external CDNs. Production must ensure visitors receive only the intended scripts and assets; externally hosted executable code should be pinned or otherwise integrity-protected so a compromised upstream source cannot silently alter the page.

### Information Disclosure

The current production app does not process user accounts or personal data, but server-side configuration and future database-backed responses remain sensitive. Production responses and logs must not expose secrets, cookies, authorization headers, stack traces, or database connection details. Public endpoints should continue returning only minimal information, as the current health check does.

### Denial of Service

Both `/` and `/api/healthz` are public. Even with a small surface, production should avoid unbounded request parsing, heavy unauthenticated computation, or future endpoints that allow attackers to amplify traffic or tie up server resources. Large static assets and future API routes should be reviewed for abuse potential if the app expands.

### Elevation of Privilege

There is no production authentication or role system today, so classic privilege-escalation issues are not yet present. The security guarantee for future growth is that any new API route that reads or mutates data must enforce server-side authorization and use parameterized database access; client-side controls alone must never be trusted.
