# Call Tracking Attribution — Integration Plan (CallRail + WhatConverts)

> Planning document for Atrium Flow. Goal: make phone calls a first-class,
> automatically-attributed `LeadTouch` event, with the **highest-confidence
> attribution possible** regardless of which call tracking platform a firm (or
> their marketing agency) uses. CallRail is the first provider; WhatConverts is
> the second. The design is provider-agnostic from day one so a third provider
> is a config + adapter, not a rewrite.

Status: planning / pre-development. Depends on #117 (LeadTouch table + visitor
tracking script + touch timeline UI).

---

## 1. The north star

A phone call should land in the touch timeline next to page views and form
submissions, attached to the right lead **automatically**, with the marketing
context (source / medium / campaign / keyword / landing page / click IDs)
preserved. Two confidence tiers:

- **High confidence** — the call is tied to a *known browser visitor* via a
  shared identifier (our `af_visitor_id` round-tripped through the provider's
  DNI session). This is the whole game; it's what lets us stitch a call to the
  exact ad/keyword that drove it and later close the loop to Google Ads.
- **Medium confidence** — no visitor bridge available; we fall back to matching
  the caller's phone number to a known contact.
- **Unattributed** — neither matched; parked in a review queue.

Everything below serves "maximize the share of calls that land in the high-
confidence tier, automatically."

---

## 2. Provider-agnostic core (build this once)

Both CallRail and WhatConverts converge on the same primitives: a JS snippet
that does Dynamic Number Insertion (DNI) using first-party cookies, automatic
gclid/UTM/keyword capture, a webhook that POSTs JSON when a call/lead is
recorded, and a REST API. So normalize once.

### 2.1 Normalized inbound model

Map every provider payload into one internal shape before touching the domain:

```
NormalizedCallEvent {
  Provider            // "callrail" | "whatconverts"
  ProviderCallId      // idempotency key (per provider)
  EventKind           // CallCompleted | CallUpdated | Form | Sms | Chat
  OccurredAt
  VisitorId?          // our af_visitor_id, recovered from the provider's custom-data bridge
  CallerPhone (E.164)
  TrackingNumber
  DestinationNumber
  Direction           // inbound | outbound
  FirstCall?          // new caller vs repeat
  Answered?
  Spam?
  // marketing
  Source / Medium / Campaign / Content / Keyword
  LandingPage / Referrer
  ClickIds { gclid, gbraid, wbraid, fbclid, msclkid, gaClientId }
  // qualification (provider-supplied)
  LeadStatus? / Qualified? / Value? / Tags[] / Sentiment? / Summary?
  // media
  RecordingRef?       // provider URL — requires auth to play, do NOT treat as public
  Transcript?
  Raw (JSON)          // keep the original payload for debugging/backfill
}
```

### 2.2 Provider adapter interface

```
ICallTrackingProvider {
  string Key
  bool   VerifyWebhook(HttpRequest req, TenantSettings s)   // secret/HMAC per provider
  NormalizedCallEvent Parse(JsonDocument payload)
  Task<bool> TestConnection(TenantSettings s)               // REST: validate creds
  Task<Stream> FetchRecording(string recordingRef, TenantSettings s) // auth proxy
}
```

The webhook controller, lead-resolution logic, `LeadTouch`/`LeadActivity`
writers, idempotency, and UI are all **provider-neutral**. Only `Parse`,
`VerifyWebhook`, and the credential set differ per provider.

> Per the original ticket: don't build the abstraction *speculatively*. But we
> now know there's a confirmed second provider with the same shape, so building
> the thin interface above with two implementations is justified — it is not
> premature.

### 2.3 Webhook endpoint (shared)

`POST /api/integrations/call-tracking/{provider}/webhook/{tenantId}`

- Anonymous (providers can't present a JWT). Trust comes from (a) the
  unguessable `{tenantId}` segment + per-tenant secret and (b) provider
  signature verification where supported.
- **Return `200` fast, process async** (enqueue → worker). Both providers retry
  on non-2xx, so the same event *will* arrive more than once.
- Rate-limited per tenant.

### 2.4 Lead resolution (shared priority order)

1. **VisitorId match (high confidence).** If the bridge recovered our
   `af_visitor_id`, attach to leads with that `VisitorId`. **Backfill** any
   prior anonymous `LeadTouch` rows for the same visitor to the resolved lead.
2. **Phone match (medium confidence).** Normalize to E.164, match
   `Contact.Phone`. Treat as *genuinely* lower confidence — at a law firm,
   spouses/family share numbers and "attach to most recent non-terminal lead"
   can mis-stitch. Prefer routing ambiguous matches to the review queue over
   auto-attaching. Stamp `Metadata.match_method = "phone"`.
3. **No match → unattributed.** Write the touch with `LeadId = NULL`, surface in
   an "Unattributed calls" admin view (manual-link UI is a follow-up).

Always record `match_method` + `match_confidence` on the touch so reporting can
separate DNI-bridged from phone-only attribution.

### 2.5 Idempotency = upsert, not ignore

Key on `(Provider, ProviderCallId)`. Later "updated" events (transcript, tags,
final value) must **enrich the same row**, not be discarded. "Replaying doesn't
duplicate" is necessary but insufficient — model it as upsert-on-update.

### 2.6 Recordings are not public links

Both providers return recording URLs that **require the account's API
credentials to play**. Never render the raw URL as an anchor. Proxy playback
through our backend with the tenant's key — which also gives us the **access
audit trail** the privacy section requires, for free. Honor a per-tenant
"store metadata only, suppress recording/transcript" toggle (transcripts carry
privileged content too, not just recordings).

---

## 3. CallRail (Provider #1) — revised from the original ticket

The original ticket is directionally right but has three assumptions that must
change before development. Verified against CallRail docs (May 2026).

### 3.1 CORRECTION — the visitor-ID bridge

The ticket assumed `CallTrk.setExternal({ external_id: af_visitor_id })`.
**That JS method does not exist publicly.** CallRail's actual, documented
mechanism is **Custom Cookie Capture**:

- You register cookie name(s) in the firm's CallRail account; CallRail's own JS
  snippet reads those **first-party cookies** during the visitor session and
  returns them in the webhook under a `custom` object (key/value pairs).
- Knock-on: our embed writes `af_visitor_id` to **localStorage**. CallRail can
  only see **cookies**. So the bridge becomes:
  1. Embed/tracking script **mirrors `af_visitor_id` into a first-party cookie**
     on the firm's domain.
  2. Tenant onboarding **registers that cookie name** in CallRail Custom Cookie
     Capture (manual in their UI, or via CallRail API if we automate it).
  3. Webhook handler reads `payload.custom.af_visitor_id`.
- Requires **website-pool visitor tracking** (CallRail JS snippet + number
  pool). Static/source-level numbers won't carry it → those tenants get
  phone-match only.

This is *more* robust than a JS shim (documented, survives CallRail JS changes)
and the same channel can carry click IDs.

### 3.2 CORRECTION — one webhook isn't enough

CallRail's real events: **pre-call, post-call, call-routing-complete,
call-modified, outbound-post-call.** The ticket's `call_completed` only carries
duration/recording. **Transcript, tags, call score, lead status, and call value
are added later and only arrive via `call-modified`.** Subscribe to **post-call
+ call-modified** and upsert by `call_id`.

### 3.3 CORRECTION — webhook auth

Verify whether CallRail actually signs (HMAC header) before building signature
verification. Historically the pattern is an **unguessable secret in the
per-tenant webhook URL**, not a signed header. The `/{tenantId}` + secret path
is the right instinct.

### 3.4 Tenant config (`TenantSettings`)

`CallRailApiKey` (encrypted, `CalendarTokenProtector` pattern),
`CallRailAccountId`, `CallRailWebhookSecret`, `CallRailEnabled`. Settings UI
under **Settings → Integrations → Call Tracking** with a **test-connection**
button (uses CallRail **REST API v3** to validate key/account + list numbers).

### 3.5 Payload → `NormalizedCallEvent`

```
ProviderCallId = payload.id / call_id
EventKind      = post-call | call-modified
OccurredAt     = payload.start_time
VisitorId      = payload.custom.af_visitor_id      // via Custom Cookie Capture
CallerPhone    = payload.customer_phone_number
ClickIds       = gclid / gbraid / wbraid / fbclid / msclkid / ga
Source/Medium/Campaign/Keyword/LandingPage/Referrer = direct
FirstCall/Answered/Spam = first_call / answered / (spam flag)
Value/Tags/LeadStatus/Transcript = (mostly from call-modified)
RecordingRef   = payload.recording (auth-required)
```

### 3.6 Revised acceptance criteria

- [ ] `TenantSettings` CallRail fields with encrypted-at-rest API key
- [ ] Settings UI: test-connection (REST v3) + enable/disable
- [ ] `POST /.../callrail/webhook/{tenantId}` — tenant secret verify (+ HMAC *if*
      confirmed), fast 200, async processing
- [ ] Subscribe **post-call + call-modified**; **upsert by `call_id`**
- [ ] Bridge: embed mirrors `af_visitor_id` to a **first-party cookie**;
      onboarding registers it in CallRail Custom Cookie Capture
- [ ] Map payload → `LeadTouch` (EventType=Call) + `LeadActivity` (Call)
- [ ] Capture **click IDs**, not just UTMs
- [ ] Resolution: VisitorId → phone → unattributed; backfill on VisitorId match
- [ ] Recording surfaced via **authed backend proxy** (with access audit), not
      raw URL; honor suppress-recording/transcript toggle
- [ ] Idempotent across retries *and* modifications

---

## 4. WhatConverts (Provider #2) — research

WhatConverts maps onto the same core with **two notable advantages** over
CallRail. Verified against WhatConverts docs (May 2026).

### 4.1 ADVANTAGE — a real JS API for the bridge

WhatConverts lets you attach your own data to the visitor/lead via its tracking
JavaScript (Capture Custom Data / Event Tracking additional fields), e.g.
conceptually:

```javascript
// attach our visitor id to the WhatConverts visitor profile
whatconverts.capture_data({ external_id: af_visitor_id });
```

- Attaches to the **visitor profile**, persists across the session, and flows
  into **any lead** that visitor creates — appearing in the webhook payload
  (`custom_fields` / captured-data object) and the REST API.
- This is exactly the bridge CallRail lacks. No cookie-registration step in the
  provider UI; our tracking script just calls it when `window.whatconverts` is
  present. Cleaner and higher-confidence than CallRail's cookie capture.
- **Verify the exact method name + payload key** against the live "Capture
  Custom Data" / "JavaScript API" docs (support pages block automated fetch).
- Still requires a **dynamic number pool** (DNI) for visitor-level tracking.

### 4.2 ADVANTAGE — unified, lead-centric webhook

WhatConverts fires **one webhook on "lead created or updated"** with a `trigger`
field and a single lead object; `lead_type` distinguishes
**call / form / chat / text**. So one adapter handles *all* intake channels —
not just calls — with no extra endpoints. The `trigger` (created vs updated)
maps directly to our upsert semantics.

Built-in **lead analysis** is richer and already surfaced in the payload:
`quotable`, `quote_value`, `sales_value`, `lead_score`, `lead_state`, plus
**Keyword / Intent / Sentiment / Topic detection** and `lead_summary`. Strong
material for automatic lead qualification (section 6).

### 4.3 Webhook security & API

- **Webhook security:** WhatConverts supports a signing secret / signature
  (HMAC-style) on outbound webhooks. *Verify exact header name + algorithm on
  the live "Webhook Security" doc before implementing.*
- **REST API:** HTTP **Basic Auth** with **API Token (username) + API Secret
  (password)**. Use for test-connection, recording fetch, and reconciliation.

### 4.4 Payload → `NormalizedCallEvent`

```
ProviderCallId = lead_id
EventKind      = trigger (created → CallCompleted, updated → CallUpdated)   [lead_type=phone]
OccurredAt     = date_created
VisitorId      = custom_fields.external_id           // via capture-custom-data
CallerPhone    = caller_number
Tracking/Dest  = tracking_number / destination_number
Source/Medium/Campaign/Content/Keyword = lead_source / lead_medium / lead_campaign / lead_content / lead_keyword
LandingPage/Referrer = landing_url / lead_url
ClickIds       = gclid / utm_* (auto-captured)
Qualification  = quotable / quote_value / sales_value / lead_score / lead_state
Analysis       = intent / sentiment / topic / lead_summary
Call fields    = call_status / call_duration_seconds / phone_name
Device         = operating_system / browser / device_type
```

### 4.5 WhatConverts-specific notes

- `lead_type` lets us light up **forms, chats, and SMS** as `LeadTouch` events
  with near-zero extra work — broader than the CallRail scope.
- Because qualification fields are first-class, WhatConverts tenants can get
  auto-lead-scoring on day one without us building NLP.

---

## 5. CallRail vs WhatConverts — side by side

| Capability | CallRail | WhatConverts |
|---|---|---|
| DNI / visitor tracking | Number pool + JS snippet | Number pool + JS snippet |
| **Push our visitor ID** | Custom **Cookie Capture** (register cookie name; we mirror id to a 1st-party cookie) | **Capture Custom Data** JS call (`external_id`) — cleaner |
| Webhook model | Multiple events (post-call, **call-modified**, …) | **Single** lead webhook, `trigger` + `lead_type` |
| Channels in one pipe | Calls (forms/SMS separate) | **Calls + forms + chat + SMS** unified |
| Auto gclid/UTM/keyword | Yes | Yes |
| Built-in qualification | Conversation Intelligence (via call-modified) | **First-class** (quotable, score, intent, sentiment) |
| Webhook auth | URL secret (verify if HMAC) | Signing secret / signature (verify exact mech) |
| REST API auth | API key | Basic auth (token + secret) |
| Recording access | Auth-required URL | Auth-required URL |

**Implication:** the provider-agnostic core in §2 is correct, and WhatConverts
actually *reduces* work (one webhook, native qualification, built-in JS bridge).
The hardest provider is CallRail (cookie bridge + multi-event upsert), so
building CallRail first de-risks the abstraction.

---

## 6. Higher-leverage uses (beyond "log the call")

Ranked by impact on accurate, automatic attribution:

1. **Closed-loop offline conversion export (highest ROI).** Persist the
   **click IDs** on every call touch. When call → lead → signed client, push the
   conversion + case value back to Google Ads / GA4. Tells Google which
   *keywords actually produce signed cases*, not just calls. Both providers
   capture gclid; this is the entire reason the visitor bridge matters.
2. **Automatic lead qualification.** Ingest provider qualification
   (WhatConverts: quotable/score/intent/sentiment; CallRail: Conversation
   Intelligence via call-modified) to auto-classify practice area, set a lead
   score, and **filter spam/sales calls out of attribution** so reporting is
   clean.
3. **Spam + first-call filtering.** Use spam flag + first-call/repeat to avoid
   attributing junk or existing-client callbacks as new acquisition, and to
   avoid duplicate-lead creation.
4. **All intake channels through one pipe.** WhatConverts' unified webhook (and
   CallRail forms/SMS) means forms/chat/text become `LeadTouch` rows with the
   same adapter — broader timeline coverage, little extra cost.
5. **Real-time intake screen-pop.** CallRail pre-call event enables "incoming
   call — known lead from Google Ads campaign X" once the bridge exists. Keep as
   a fast-follow, not "out of scope."
6. **Reconciliation safety net.** Periodically pull leads/calls via each REST API
   to catch dropped webhooks and enrich number↔campaign mapping. Self-healing.

---

## 7. Open questions to verify at the machine

- **CallRail:** exact field name/nesting of the `custom` cookie object in a real
  post-call payload; whether CallRail signs webhooks (HMAC) or relies on URL
  secret; whether target firms run number-pool visitor tracking (determines
  day-one high-confidence share).
- **WhatConverts:** exact JS method name + payload key for captured custom data;
  exact webhook signature header + algorithm (Webhook Security doc).
- **Both:** confirm recording/transcript endpoints + auth for the proxy; data
  residency / consent posture for two-party-consent states (CA, FL, …) —
  document the assumption that the firm owns recording consent.

---

## 8. Suggested build order

1. #117 ships (LeadTouch + tracking script + timeline UI). Hard dependency.
2. Provider-agnostic core: normalized model, `ICallTrackingProvider`, shared
   webhook endpoint, async queue, lead resolution, upsert idempotency,
   recording proxy + audit.
3. **CallRail adapter** (hardest bridge → de-risks the abstraction).
4. **WhatConverts adapter** (validates the abstraction; should be mostly
   mapping + the cleaner JS bridge).
5. Phase 2: closed-loop conversion export + qualification ingestion + unattributed
   manual-link UI + real-time screen-pop.

---

### Sources

- CallRail Webhooks — https://support.callrail.com/hc/en-us/articles/5711246459149-Webhooks
- CallRail Custom Cookie Capture — https://support.callrail.com/hc/en-us/articles/5711577152909-Custom-Cookie-Capture
- CallRail Building a custom integration — https://support.callrail.com/hc/en-us/articles/5712041065229-Building-a-custom-integration
- CallRail API v3 — https://apidocs.callrail.com/
- WhatConverts Webhooks (phone call data) — https://www.whatconverts.com/help/docs/automations-and-advanced-systems/webhooks/receive-phone-call-data-with-webhooks/
- WhatConverts Capture Custom Data — https://www.whatconverts.com/help/docs/automations-and-advanced-systems/capture-custom-data/
- WhatConverts Custom Fields — https://www.whatconverts.com/help/docs/using-whatconverts/lead-manager/custom-fields/
- WhatConverts Webhook Security — https://www.whatconverts.com/help/docs/automations-and-advanced-systems/webhooks/webhook-security/
- WhatConverts API (Leads / Overview) — https://www.whatconverts.com/api/leads/ · https://www.whatconverts.com/api/overview/
- WhatConverts DNI — https://www.whatconverts.com/help/docs/faq/how-does-dynamic-number-insertion-dni-work/
</content>
