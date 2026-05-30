# Google Ads API in Atrium Flow — Capability Strategy

**Framing:** This is a *capability* plan, not a code plan. Atrium Flow already has a codebase and database; the question here is **what are the highest-value things we can do with the Google Ads API** to help (a) our firm operate and (b) the firms/clients who use Atrium Flow win with their advertising.

**Anchor scope:** Reporting & intelligence (read-only). We pull data and turn it into insight — we do **not** create or edit ads. That keeps risk low and trust high, while leaving room for "act on it" features later.

**Date:** 2026-05-30

---

## 1. The strategic bet

Anyone can show a client their spend and clicks — Google's own UI does that for free. The value Atrium Flow adds is **turning raw Ads data into decisions and saved time**:

1. **Save the firm hours** on manual reporting and account babysitting.
2. **Catch problems before the client does** (overspend, conversion tracking broke, CPC spiked).
3. **Tell the story in plain English**, automatically, per client.

Those three themes should drive prioritization. Everything below is sorted into **Tier 1 (table stakes)**, **Tier 2 (differentiators)**, and **Tier 3 (moonshots)** against them.

---

## 2. Account model: design for agencies first

Our clients are firms that often manage **many** advertiser accounts. The integration should assume an **MCC (Manager account) model** from day one:

- **One connection, many accounts.** A firm connects its Manager account once; we enumerate all client accounts under it (`listAccessibleCustomers` + `login-customer-id`) instead of forcing a separate OAuth per advertiser.
- **Portfolio-level roll-ups** across all of a firm's accounts, plus drill-down into any single client.
- **Role-aware views:** the firm sees everything; an end-client sees only their own account (white-label).

Getting this right early is what separates a "connect your Google Ads" toy from an agency-grade platform. Also plan for **Standard developer-token access** (higher quotas) since agencies generate real volume.

---

## 3. Tier 1 — Table stakes (must-have to be credible)

| Capability | What it delivers | Google Ads API basis |
| --- | --- | --- |
| **Automated daily sync** | Hands-off; data is fresh every morning | `GoogleAdsService.SearchStream` + GAQL, daily cron |
| **Core metrics, all levels** | Spend, clicks, impressions, CPC, CTR, conversions, conv. value, CPA, ROAS | `metrics.*` on campaign / ad_group / ad_group_ad / keyword_view |
| **The full hierarchy** | Account → campaign → ad group → ad → keyword | resource-specific GAQL queries |
| **Date-range + comparisons** | WoW / MoM / YoY deltas, not just absolutes | `segments.date`, compare two windows |
| **Multi-account roll-up** | Firm sees the whole book of business at a glance | iterate accounts under MCC |
| **Currency & timezone correctness** | Numbers clients trust (micros→currency, per-account TZ) | cost in micros ÷ 1e6, account `time_zone` |
| **Rolling re-sync window** | Yesterday's number isn't final — re-pull last ~3–7 days | Ads cost/conversions adjust retroactively |

**Why it matters:** without these, nobody trusts the tool. The "rolling re-sync" detail is the one teams most often get wrong — Google keeps adjusting recent cost and (especially) conversions for days, so a one-shot daily pull silently goes stale.

---

## 4. Tier 2 — Differentiators (where Atrium Flow earns its keep)

These are the features that make a firm *pay for and keep* Atrium Flow.

### 4.1 AI narrative reporting (highest leverage — we already have Claude)
Auto-generate a plain-English performance summary per client, per period:
> *"Spend was £4,210 last week (+12% WoW). CPA improved 8% to £31, driven mainly by the 'Brand – Exact' campaign. Conversions dipped Thursday — likely the budget cap on 'Generic – Broad', which lost 34% impression share to budget."*

This collapses an hour of analyst write-up into seconds and is the single most differentiating feature. Feed the synced metrics + deltas + impression-share signals to Claude and template the output. Ties directly to "tell the story in plain English."

### 4.2 Budget pacing & overspend alerts
Track each campaign/account against its monthly budget, **project end-of-month spend** from run-rate, and alert when pacing is off (over or under).
- API basis: `campaign_budget.amount_micros`, daily spend, simple projection.
- Why: overspend is the #1 thing that loses an agency a client. Catching it first is gold.

### 4.3 Anomaly detection & alerting
Daily scan for: spend spikes/drops, CPC surges, CTR collapse, **conversions suddenly zero** (broken tracking), impression-share cliffs. Push to in-app / email / Slack.
- Why: "catch problems before the client does." A zero-conversion alert alone justifies the product.

### 4.4 Wasted-spend / search-terms intelligence
Surface the actual search queries draining budget and flag negative-keyword opportunities.
- API basis: `search_term_view` with cost/conversions; rank by spend-with-no-conversions.
- Why: directly saves clients money — the easiest ROI story to tell.

### 4.5 Impression share & competitive context
Show where accounts are **losing impressions to budget vs. to rank**, and absolute/top impression share.
- API basis: `metrics.search_impression_share`, `search_budget_lost_impression_share`, `search_rank_lost_impression_share`, `search_absolute_top_impression_share`.
- Why: turns "we're doing fine" into "here's exactly the headroom and what's capping it."

### 4.6 Google's own Recommendations, surfaced (read-only)
Pull Google's optimization recommendations and present them as a prioritized to-do list (estimated impact included) — without auto-applying.
- API basis: `RecommendationService` / `recommendation` resource.
- Why: high-value optimization queue with zero analyst research time.

### 4.7 Conversion-tracking health monitor
Continuously verify conversions are firing and attribution looks sane; alert on drift or zeros.
- Why: broken tracking silently wrecks reporting and optimization; being the system that catches it builds deep trust.

### 4.8 Change history ↔ performance correlation
Log account changes and overlay them on the performance timeline ("CPA jumped the day bids changed").
- API basis: `change_event` resource.
- Why: explains *why* numbers moved — exactly what clients ask in every meeting.

### 4.9 Scheduled, white-label client reports
Auto-generate branded PDF/email reports (powered by the AI narrative + charts) on a schedule.
- Why: the deliverable agencies currently build by hand every month. Automating it is pure time saved.

---

## 5. Tier 3 — Moonshots (roadmap / moats)

| Capability | Value |
| --- | --- |
| **Cross-channel blending** | Unify Google Ads with **GA4** (Data API), Search Console, Meta, etc. into one ROI view — clients live across channels |
| **Portfolio benchmarking** | Compare a client's CPC/CTR/CPA against the firm's own portfolio or industry norms |
| **Forecasting** | Project spend/conversions and run budget scenarios (KeywordPlan / forecast metrics) |
| **PMax & asset intelligence** | Asset-group and RSA asset-level performance, creative strength over time (`asset`, `asset_group`) |
| **Geo heatmaps** | Performance by location for local clients (`geographic_view`) |
| **"Act on it" (write) — opt-in** | Eventually apply negatives / pause wasters / accept recommendations from inside Atrium Flow (separate trust + scope decision) |
| **Conversational analytics** | "Why did CPA go up last week?" answered by Claude over the synced dataset |

---

## 6. Value map — who benefits from what

**For our firm (operations):**
- Portfolio overview + an **"accounts needing attention" triage queue** (anomalies, pacing, broken tracking) → analysts work by exception, not by spreadsheet.
- Automated reporting → hours back per client per month.
- Early-warning alerts → fewer fire drills and churn.

**For our clients (the firms using Atrium Flow):**
- Plain-English insights they actually understand.
- Proof of ROI (ROAS/CPA trends, wasted-spend recovered).
- Branded reports they can forward to *their* stakeholders.

---

## 7. Data realities to design around (so the numbers are trustworthy)

- **Cost is in micros** — divide by 1,000,000; respect each account's currency.
- **Recent data isn't final** — cost settles within hours, conversions for days (view-through longer). Always re-sync a rolling window.
- **Per-account timezone** — "yesterday" differs by account; segment dates in the account's TZ.
- **Quotas** — request **Standard** developer-token access for agency volume; use `SearchStream`, batch by account, back off on `RESOURCE_EXHAUSTED`.
- **Cache, don't re-query** — store synced daily metrics in our DB so dashboards are instant and we stay well under API limits.
- **Read-only by discipline** — only `SELECT` GAQL; no mutate calls until/unless we deliberately add a write tier.

---

## 8. Compliance & trust (non-negotiables)

- **Developer token + OAuth client secret stay server-side**; refresh tokens encrypted at rest.
- **OAuth consent screen verification** for production external users (has lead time — start early).
- **Google API Services User Data Policy**: clear privacy disclosure, working disconnect/delete path, strict per-tenant data isolation.
- **Least privilege**: reporting scope, read-only behavior, audited sync jobs (never log tokens).

---

## 9. Recommended prioritization (build order)

1. **Foundation:** MCC OAuth + multi-account sync + Tier 1 metrics, cached daily with rolling re-sync. *(Trust + table stakes.)*
2. **First wow:** AI narrative reporting (§4.1) + anomaly/zero-conversion alerts (§4.3) + budget pacing (§4.2). *(Immediate, demoable differentiation using strengths we already have.)*
3. **Money story:** wasted-spend/search terms (§4.4) + impression share (§4.5) + Recommendations (§4.6).
4. **Stickiness:** scheduled white-label reports (§4.9) + change-event correlation (§4.8) + conversion-health (§4.7).
5. **Moats:** cross-channel blending and benchmarking (Tier 3).

The thread: **earn trust with rock-solid data, then win on AI insight + proactive alerting + automated reporting** — the three things that save the firm time and make clients feel looked after.

---

## 10. Open questions to sharpen the plan

1. **Agency vs. direct:** are Atrium Flow's users mostly agencies managing many accounts (→ MCC-first), single-business advertisers, or both?
2. **AI depth:** how far do we push AI narratives — summaries only, or conversational "ask anything about my account"?
3. **Alert channels:** in-app only, or email/Slack/SMS too?
4. **Write-back appetite:** is "act on it" (apply negatives, accept recommendations) on the roadmap, or strictly read-only long-term?
5. **Cross-channel:** is unifying GA4/Meta/etc. a near-term expectation or a later moat?
6. **MCC/token status:** do we already hold a Manager account + production developer token, or do we start that application now (it gates go-live)?
