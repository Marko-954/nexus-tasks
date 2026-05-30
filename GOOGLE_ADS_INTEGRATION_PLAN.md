# Google Ads API Integration — Atrium Flow

**Status:** Planning / design
**Scope:** Per-user, **reporting only** (read ads, daily cost, CPC, clicks, impressions). No ad creation or editing.
**Date:** 2026-05-30

---

## 1. TL;DR — the one decision that drives everything

Atrium Flow today is a **single static `index.html`**: client-side React (Babel-standalone in the browser), no backend, no database, and not even `localStorage` — task state lives only in memory. Its Google Calendar integration works because Calendar can be called **directly from the browser** with a short-lived OAuth token from Google Identity Services (GIS).

**Google Ads cannot be done that way.** Three hard constraints make a browser-only approach impossible:

1. **Developer token must stay secret.** Every Google Ads API call requires a *developer token* tied to a Manager (MCC) account. If we ship it in `index.html`, anyone can read it and it gets revoked. It must live server-side.
2. **No browser CORS / SDK path.** The Ads API is gRPC-first (REST exists but is not CORS-enabled for browser apps). You call it from a server using the official client libraries.
3. **"Automatically pull down on a daily basis" requires offline access.** Daily unattended syncs need an OAuth **refresh token** (offline access). The current GIS browser flow only issues short-lived access tokens with no refresh token, and only while a tab is open. A backend with a scheduler is required.

**Conclusion:** This feature requires standing up a small **backend service** (API + database + scheduler) alongside the existing static frontend. The bulk of this plan is about that backend. The frontend work is comparatively small: an OAuth "Connect Google Ads" button and a reporting view.

> If a backend is genuinely off the table, the only alternatives are (a) a fully managed third-party connector (e.g. an ETL/reporting vendor) or (b) reusing an existing internal PPC platform if one already exists (see §11). Both still terminate in a database the frontend reads from — there is no pure-frontend version of this.

---

## 2. Current-state assessment

| Area | Today | Implication for Google Ads |
| --- | --- | --- |
| Hosting | Static `index.html`, React via CDN + Babel-standalone | Fine to keep for the UI; needs a backend peer |
| Persistence | None (in-memory) | Must add a DB to store tokens + cached metrics |
| Auth model | No user accounts; per-device, settings typed into UI | **Need real user accounts** to key connections per user |
| Google OAuth | GIS browser token client, scope `calendar.events`, no refresh token (`index.html:264-342`) | Need server-side OAuth **authorization-code** flow with `access_type=offline` |
| Secrets | User pastes their own Claude key / GCP client ID into the UI | Developer token + OAuth client secret must move server-side |
| Scheduling | None | Need a cron/worker for daily pulls |

The existing GIS code (`useGoogleAuth`) is a good reference for the *consent UX*, but the token flow itself is not reusable for Ads.

---

## 3. Target architecture

```
┌─────────────────┐     1. "Connect Google Ads"     ┌──────────────────────┐
│  Atrium Flow     │ ──────────────────────────────▶ │  Atrium Flow Backend │
│  (static SPA)    │                                  │  (API + scheduler)   │
│                  │ ◀──── 5. reporting JSON ──────── │                      │
└─────────────────┘                                  └─────────┬────────────┘
        ▲                                                       │
        │ 4. render charts/tables                               │ 2. OAuth code flow (offline)
        │                                                       │ 3. daily GAQL pulls
        │                                              ┌────────▼────────────┐
        └──────────────────────────────────────────── │   Google Ads API    │
                                                       │   (v19, GAQL)       │
                                                       └─────────────────────┘
                                                       ┌─────────────────────┐
                                            store ───▶ │  DB: users, oauth    │
                                                       │  connections, daily  │
                                                       │  ad metrics cache    │
                                                       └─────────────────────┘
```

**Components to build:**

1. **Backend API service** — recommended **Node.js + TypeScript** (reuses the team's JS skillset and the official `google-ads-api` npm library) or **Python + FastAPI** (official `google-ads` library). Either is fine; pick by team familiarity. Hosting: any container/PaaS (Cloud Run, Render, Fly, Railway).
2. **Database** — Postgres. Stores users, encrypted OAuth refresh tokens, connected Ads customer IDs, and a cache of daily metrics so the UI is fast and we don't hit Ads API rate limits on every page load.
3. **Scheduler/worker** — a daily cron job that iterates connected accounts and pulls yesterday's metrics (plus a rolling re-sync window for late-attributed conversions/cost adjustments).
4. **Frontend additions** — a "Connect Google Ads" flow and a reporting view in `index.html` (or a migration to a small build setup — see §10).

---

## 4. Google Ads API prerequisites (one-time, ours to own)

These are platform-level and must be done before any user can connect:

1. **Manager (MCC) account** — create one if Atrium Flow doesn't have it.
2. **Developer token** — apply in the MCC under *API Center*. Starts at **Test access** (only works against test accounts), then apply for **Basic access** (production, ~15k operations/day), then **Standard** if volume grows. Apply early; approval can take days.
3. **Google Cloud project + OAuth client** — type **Web application**, with our backend's redirect URI (e.g. `https://api.atriumflow.com/oauth/google-ads/callback`). This gives the **client ID + client secret** (secret stays server-side).
4. **OAuth consent screen** — configure scopes, branding, and submit for **verification** (required for sensitive scopes used in production with external users). Budget time for Google's review.

**OAuth scope needed:** `https://www.googleapis.com/auth/adwords` (full Ads scope — there is no read-only variant; we enforce read-only by only ever issuing reporting queries).

---

## 5. Per-user connection flow

Each of the ~15–20 users who opts in connects *their own* Google Ads account:

1. User clicks **Connect Google Ads** in Atrium Flow.
2. Backend redirects to Google's OAuth consent URL with `access_type=offline` and `prompt=consent` (forces a refresh token to be returned).
3. User grants access; Google redirects back to our callback with an authorization `code`.
4. Backend exchanges the code for an **access token + refresh token**, and stores the **refresh token encrypted** (e.g. envelope encryption / KMS, or at minimum AES-GCM with a key from secrets manager) keyed to the user.
5. Backend calls `CustomerService.ListAccessibleCustomers` to enumerate the Ads accounts the user can see, then lets the user pick which account(s) to report on. Store the selected **customer IDs** (and, if behind an MCC, the `login-customer-id`).
6. Connection now shows as "Connected" with the account name/ID — mirroring the existing Calendar "connected" UI (`index.html:442-454`).

**Disconnect** = revoke the token with Google and delete stored tokens/cache for that user (mirrors `gcal.disconnect`).

---

## 6. What we pull (data model)

For reporting we only need a handful of GAQL queries. Cost fields come back in **micros** (divide by 1,000,000).

**A. Daily campaign performance** (the core "cost per day" view):
```sql
SELECT campaign.id, campaign.name, campaign.status,
       segments.date,
       metrics.cost_micros, metrics.clicks, metrics.impressions,
       metrics.average_cpc, metrics.ctr, metrics.conversions
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
```

**B. Ad-level detail** ("the ads they're running"):
```sql
SELECT ad_group_ad.ad.id,
       ad_group_ad.ad.name,
       ad_group_ad.ad.type,
       ad_group_ad.status,
       ad_group.name, campaign.name,
       segments.date,
       metrics.cost_micros, metrics.clicks, metrics.impressions,
       metrics.average_cpc
FROM ad_group_ad
WHERE segments.date DURING LAST_30_DAYS
```

Use `GoogleAdsService.SearchStream` for these (efficient for large result sets).

**Suggested tables:**

```
users(id, email, ...)                         -- real accounts, replacing today's device-local model
ad_connections(id, user_id, customer_id,
               login_customer_id, account_name,
               refresh_token_enc, status,
               connected_at, last_synced_at)
ad_daily_metrics(connection_id, date,
                 campaign_id, campaign_name,
                 ad_id NULL, ad_name NULL,
                 cost_micros, clicks, impressions,
                 avg_cpc_micros, conversions,
                 UNIQUE(connection_id, date, campaign_id, ad_id))
```

Storing the daily cache means reports load instantly and we control API usage.

---

## 7. Sync & scheduling strategy

- **Daily job** (e.g. 06:00 in each account's timezone, or a single UTC run): for every active connection, pull **yesterday** plus a **rolling 3-day re-sync** (Google adjusts cost/conversions retroactively), and upsert into `ad_daily_metrics`.
- **On-connect backfill:** when a user first connects, pull the last **30–90 days** so the UI isn't empty.
- **Token refresh:** access tokens expire (~1h); the client library auto-refreshes using the stored refresh token. Handle `invalid_grant` (user revoked/expired) by flagging the connection as "Reconnect needed" and surfacing that in the UI.
- **Rate limits / quotas:** Basic access ≈ 15,000 operations/day. With ~20 accounts and a couple of queries each per day, we're far under budget. Add retry-with-backoff on `RESOURCE_EXHAUSTED`.
- **Idempotency:** upserts keyed on `(connection_id, date, campaign_id, ad_id)` so re-runs are safe.

---

## 8. Reporting UI (frontend)

Add to Atrium Flow, styled to match the existing neon/dark theme (CSS in `index.html:12-148`):

- **Settings → Connections:** a "Google Ads" card next to the Calendar card, reusing the connected/disconnected pattern (`SettingsPanel`, `index.html:365-472`).
- **Reports view:** a new top-nav tab beside "Tasks"/"Completed" (`Header`, `index.html:346-363`) showing:
  - KPI tiles: total spend, total clicks, avg CPC, impressions (reuse `StatsBar`, `index.html:599-620`).
  - A daily spend / CPC line chart (add a small chart lib, e.g. Chart.js via CDN to stay consistent with the current no-build approach).
  - A table of campaigns/ads with cost, clicks, CPC, sortable.
  - Date-range selector (last 7 / 30 / custom).
- All data comes from **our backend** (`GET /api/ads/metrics?range=30d`), never directly from Google.

---

## 9. Security, privacy & compliance

- **Secrets server-side only:** developer token and OAuth client secret never reach the browser. (Today the app has users paste their own keys into the UI — that pattern must NOT be extended to Ads.)
- **Refresh tokens encrypted at rest**, decrypted only in-memory at sync time; use a secrets manager / KMS for the encryption key.
- **Least privilege & read-only behavior:** only issue `SELECT` GAQL reporting queries; never mutate.
- **Google API Services User Data Policy:** because we store Google user data, the OAuth consent screen needs verification and we should have a clear privacy disclosure and a working disconnect/delete path.
- **Per-user isolation:** every query and cache row is scoped by `user_id`; never let one user read another's metrics.
- **Audit/logging:** log sync runs and failures; don't log tokens.

---

## 10. Frontend build consideration

The current single-file, in-memory, no-`localStorage` setup is fine for a demo but is the weakest link for a real per-user product (state is lost on refresh; there are no accounts). Recommended, in order:

1. **Minimum:** keep `index.html` as-is for the UI but add real user auth (session with the backend) and have the backend hold all state.
2. **Better (recommended soon):** migrate the frontend to a small Vite + React build so we get a real OAuth session, routing, and the chart lib without CDN/Babel-in-browser. This is a modest lift and pays off immediately for the reporting views.

---

## 11. Reuse check before building

Before building a backend from scratch, confirm whether the team already operates a **PPC / Google Ads sync platform** (the `duodigital.io` org appears to run marketing tooling). If an internal service already holds Google Ads OAuth connections and synced campaign/ad metrics, the cheapest path is:

- Have that platform expose a read API (per-user campaigns + daily metrics), and
- Build **only** the Atrium Flow reporting UI + a thin proxy against it.

That collapses most of §3–§7 into "consume an existing API." Worth a 30-minute check before committing to net-new infrastructure.

---

## 12. Phased rollout

| Phase | Deliverable | Notes |
| --- | --- | --- |
| 0 | Apply for developer token + create OAuth client + consent screen | **Start now** — external approvals have lead time |
| 1 | Backend skeleton: user accounts, DB, OAuth connect/callback, token storage | Foundation |
| 2 | Connect flow end-to-end against a **test** Ads account; `ListAccessibleCustomers` + account picker | Validate auth |
| 3 | GAQL reporting queries + on-connect backfill + `ad_daily_metrics` cache | Core data |
| 4 | Daily scheduler + rolling re-sync + reconnect handling | Automation |
| 5 | Frontend: Connect card + Reports view (KPIs, chart, table) | User-facing |
| 6 | Hardening: encryption, quota/backoff, audit, privacy/disconnect, consent verification | Production-ready |

---

## 13. Key decisions needed from you

1. **Backend stack** — Node/TypeScript (`google-ads-api`) or Python/FastAPI (`google-ads`)? (Recommend Node to match existing JS.)
2. **Reuse vs. build** — is there an existing internal PPC platform we should read from instead of building sync (§11)?
3. **Hosting** — where should the backend + Postgres live (Cloud Run, Render, Fly, Railway, existing infra)?
4. **Frontend** — keep single-file `index.html`, or migrate to a Vite build now (§10)?
5. **Auth** — how do users sign in to Atrium Flow itself today/going forward? (Reporting must be keyed to a real user identity.)
6. **MCC status** — do we already have a Manager account + developer token, or do we start that application now?

---

## 14. Rough effort estimate (engineering)

Assuming we build the backend (not reuse):

- Phase 0 (approvals): ~0 dev effort, **days–weeks of waiting** — start immediately.
- Phases 1–4 (backend, OAuth, sync): ~2–3 weeks for one engineer.
- Phase 5 (frontend reporting): ~1 week (more if migrating to Vite).
- Phase 6 (hardening/compliance): ~3–5 days.

**~4–5 weeks** to a production-ready, per-user reporting integration — pending Google's token/consent approvals, which gate go-live.
