# SEM Agentic Paid-Media Workflow — n8n MVP Build Specification

This operationalizes the architecture review into something you can actually build: node-by-node n8n workflow, exact API calls, deterministic formulas, JSON schemas, the LLM Strategist prompt, and the approval/execution logic. It implements the **Version 1 MVP** scope from that review:

- **3 diagnoses only:** CPA deterioration, search-term waste, conversion-rate deterioration
- **5 recommendation types only:** add negative keywords, promote keywords, adjust budget, recommend ad test, investigate landing page
- **Every action requires human approval.** Nothing auto-executes in v1 (Tier 2 "safe automation" is a v2 upgrade once the Optimization Ledger shows a track record).

Everything after Section 12 (auction insights, ad copy generation, automated execution, CRM revenue join, Postgres ledger) is flagged as **v2+** so you don't try to build the whole diagram on day one.

> **Version note:** Google Ads API versions now deprecate roughly every 12 months on a monthly release cadence (v23 shipped Jan 2026, v19 sunset Feb 2026). Endpoints below use `v23` — confirm the current version at `developers.google.com/google-ads/api/docs/release-notes` before building, since it will have moved on by the time you implement this.

---

## 1. Prerequisites & Credentials

| # | Requirement | Where to get it | n8n credential type |
|---|---|---|---|
| 1 | Google Ads Developer Token | Google Ads UI → Tools → API Center (Basic access is enough for one account; Standard needed for multiple client accounts) | stored as header value, see §2 |
| 2 | Google Cloud OAuth2 Client (Client ID/Secret) | Google Cloud Console → APIs & Services → Credentials, with the Google Ads API enabled | `Google Ads OAuth2 API` or generic `OAuth2 API` credential |
| 3 | Manager account `login-customer-id` | Your MCC account ID (only needed if querying client accounts under a manager account) | passed as header, see §2 |
| 4 | Target `customer_id` per client | Client's Google Ads account ID (no dashes) | stored in the Client Config sheet (§3.1) |
| 5 | GA4 property ID + OAuth2/service account with `analytics.readonly` scope | Google Analytics Admin → Property Settings | `Google Analytics OAuth2 API` credential |
| 6 | LLM API key | Anthropic (or OpenAI) console | `Anthropic API` credential (or HTTP Header Auth) |
| 7 | Slack bot token + channel | Slack App with `chat:write`, `chat:write.public`, interactivity enabled | `Slack API` credential |
| 8 | Google Sheets (or Postgres for v2) | Google Cloud service account with Sheets API access | `Google Sheets OAuth2 API` credential |

### n8n's native Google Ads node

n8n's built-in Google Ads node **only supports "Get all campaigns" / "Get a campaign."** It does not expose search-term reports, keyword reports, change events, mutates, or GAQL. For everything else in this spec, use the **HTTP Request node** with **Authentication → Predefined Credential Type → Google Ads API**, which lets n8n handle the OAuth2 token refresh while you supply the developer-token header manually.

---

## 2. Google Ads HTTP Request Node — Standard Config

Every Google Ads call in this workflow is an HTTP Request node configured like this:

```
Method: POST
URL: https://googleads.googleapis.com/v23/customers/{{ $json.customer_id }}/googleAds:searchStream
Authentication: Predefined Credential Type → Google Ads API
Headers:
  developer-token: {{ $credentials.developerToken }}
  login-customer-id: {{ $json.manager_customer_id }}   // omit if not under an MCC
  Content-Type: application/json
Body (JSON):
{
  "query": "{{ $json.gaql_query }}"
}
```

Use `:search` instead of `:searchStream` if you want paginated JSON responses instead of a newline-delimited stream — simpler to parse in n8n's Code node, and fine at MVP data volumes.

Mutates (writing changes back) use a different path per resource, shown in §11.

---

## 3. Data Model

### 3.1 Client Strategy Config (Google Sheet: `Clients`)

One row per client. This is what lets the agent ask "is $72 CPA acceptable *for this client*" instead of applying a generic threshold.

| Column | Type | Example |
|---|---|---|
| `client_id` | text | `carpet-co-appleton` |
| `client_name` | text | `Appleton Carpet Co.` |
| `google_ads_customer_id` | text | `123-456-7890` |
| `manager_customer_id` | text | `111-222-3333` |
| `ga4_property_id` | text | `properties/987654321` |
| `primary_objective` | text | `qualified_lead_generation` |
| `target_cpa` | number | `65` |
| `target_roas` | number | (blank if lead-gen) |
| `monthly_budget` | number | `15000` |
| `gross_margin_pct` | number | `0.42` |
| `avg_order_value` | number | (blank if lead-gen) |
| `max_auto_budget_adjustment_pct` | number | `0` *(0 = no auto-execution in v1)* |
| `max_auto_bid_adjustment_pct` | number | `0` |
| `slack_channel_id` | text | `C0123ABCD` |
| `approver_emails` | text | `owner@carpetco.com` |
| `negative_keyword_dictionary` | text (comma list) | `jobs,salary,how to,diy,tutorial,free` |
| `active` | boolean | `TRUE` |

### 3.2 Optimization Ledger (Google Sheet: `Ledger`, → Postgres in v2)

One row per recommended action, appended when generated and updated when resolved.

| Column | Type |
|---|---|
| `ledger_id` (UUID) | text |
| `date_generated` | datetime |
| `client_id` | text |
| `campaign_id` | text |
| `observation` | text |
| `diagnosis` | text |
| `action_type` | text |
| `action_detail` | text |
| `rationale` | text |
| `confidence` | text (high/medium/low) |
| `priority` | text (critical/high/medium/low) |
| `policy_tier` | text |
| `approval_required` | boolean |
| `status` | text (`pending` / `approved` / `rejected` / `modified` / `executed` / `failed`) |
| `approver` | text |
| `date_resolved` | datetime |
| `before_metrics` | JSON text |
| `after_metrics_30d` | JSON text (filled by a follow-up workflow) |
| `outcome` | text (`worked` / `no_effect` / `hurt` / `not_yet_measured`) |

v2 Postgres DDL:

```sql
CREATE TABLE optimization_ledger (
  ledger_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  date_generated TIMESTAMPTZ NOT NULL DEFAULT now(),
  client_id TEXT NOT NULL,
  campaign_id TEXT NOT NULL,
  observation TEXT,
  diagnosis TEXT,
  action_type TEXT NOT NULL,
  action_detail TEXT,
  rationale TEXT,
  confidence TEXT CHECK (confidence IN ('high','medium','low')),
  priority TEXT CHECK (priority IN ('critical','high','medium','low')),
  policy_tier TEXT,
  approval_required BOOLEAN DEFAULT TRUE,
  status TEXT DEFAULT 'pending',
  approver TEXT,
  date_resolved TIMESTAMPTZ,
  before_metrics JSONB,
  after_metrics_30d JSONB,
  outcome TEXT
);
CREATE INDEX idx_ledger_client_status ON optimization_ledger (client_id, status);
```

---

## 4. Google Ads Reports (GAQL)

All queries use `segments.date BETWEEN` for the current window and a second call for the baseline window (e.g., current = trailing 14 days, baseline = the 14 days before that, or same period prior year once you have enough history).

### 4.1 Campaign performance (current + baseline)

```sql
SELECT
  campaign.id,
  campaign.name,
  campaign.status,
  segments.date,
  metrics.impressions,
  metrics.clicks,
  metrics.cost_micros,
  metrics.conversions,
  metrics.conversions_value,
  metrics.ctr,
  metrics.average_cpc,
  metrics.search_impression_share,
  metrics.search_absolute_top_impression_share
FROM campaign
WHERE segments.date BETWEEN '2026-08-26' AND '2026-09-08'
  AND campaign.status = 'ENABLED'
ORDER BY segments.date
```

### 4.2 Search Term Report

```sql
SELECT
  search_term_view.search_term,
  search_term_view.status,
  campaign.id,
  ad_group.id,
  ad_group.name,
  segments.date,
  metrics.impressions,
  metrics.clicks,
  metrics.cost_micros,
  metrics.conversions,
  metrics.conversions_value
FROM search_term_view
WHERE segments.date BETWEEN '2026-08-26' AND '2026-09-08'
  AND campaign.id = {{campaign_id}}
ORDER BY metrics.cost_micros DESC
```

### 4.3 Keyword Performance

```sql
SELECT
  ad_group_criterion.criterion_id,
  ad_group_criterion.keyword.text,
  ad_group_criterion.keyword.match_type,
  ad_group_criterion.status,
  ad_group.id,
  campaign.id,
  metrics.impressions,
  metrics.clicks,
  metrics.cost_micros,
  metrics.conversions,
  metrics.ctr,
  metrics.average_cpc,
  metrics.search_impression_share
FROM keyword_view
WHERE segments.date BETWEEN '2026-08-26' AND '2026-09-08'
  AND campaign.id = {{campaign_id}}
```

### 4.4 Ad Performance

```sql
SELECT
  ad_group_ad.ad.id,
  ad_group_ad.status,
  ad_group_ad.ad_strength,
  ad_group.id,
  campaign.id,
  metrics.impressions,
  metrics.clicks,
  metrics.ctr,
  metrics.conversions,
  metrics.cost_micros
FROM ad_group_ad
WHERE segments.date BETWEEN '2026-08-26' AND '2026-09-08'
  AND campaign.id = {{campaign_id}}
```

### 4.5 Change Event (Change Intelligence Layer — §6 of the review)

```sql
SELECT
  change_event.change_date_time,
  change_event.change_resource_type,
  change_event.resource_change_operation,
  change_event.changed_fields,
  change_event.old_resource,
  change_event.new_resource,
  change_event.user_email,
  change_event.client_type
FROM change_event
WHERE change_event.change_date_time BETWEEN '2026-08-26 00:00:00' AND '2026-09-08 23:59:59'
ORDER BY change_event.change_date_time DESC
LIMIT 200
```

> `change_event` historically only supports a ~30-day lookback window and requires a date-range filter — confirm current limits in the docs before relying on it for longer baselines.

### 4.6 Google's Own Recommendations (§20 of the review — the "second opinion")

```sql
SELECT
  recommendation.resource_name,
  recommendation.type,
  recommendation.dismissed,
  recommendation.campaign_budget_recommendation.current_budget_amount_micros,
  recommendation.campaign_budget_recommendation.recommended_budget_amount_micros
FROM recommendation
WHERE campaign.id = {{campaign_id}}
```

`recommendation` fields are a union — you'll need one query variant per `recommendation.type` you care about (budget, keyword, ad) if you want the type-specific sub-fields.

### 4.7 GA4 — Landing Page & Conversion Signal (optional in v1, needed for §8 Landing Page Intelligence)

```
POST https://analyticsdata.googleapis.com/v1beta/properties/{{ga4_property_id}}:runReport
{
  "dateRanges": [{ "startDate": "2026-08-26", "endDate": "2026-09-08" }],
  "dimensions": [{ "name": "landingPagePlusQueryString" }, { "name": "sessionCampaignName" }],
  "metrics": [
    { "name": "sessions" },
    { "name": "conversions" },
    { "name": "engagementRate" },
    { "name": "averageSessionDuration" }
  ],
  "dimensionFilter": {
    "filter": { "fieldName": "sessionDefaultChannelGroup", "stringFilter": { "value": "Paid Search" } }
  }
}
```

---

## 5. The n8n Workflow — Node by Node

Trigger cadence: weekly for the diagnostic sweep (Monday morning), with a daily lightweight anomaly check optional once v1 is stable. Nodes are grouped by stage; each stage is a logical block you can build and test independently.

| # | Node | Type | Purpose | Key config |
|---|---|---|---|---|
| 1 | `Weekly Trigger` | Schedule Trigger | Kicks off the sweep | Cron: `0 7 * * 1` |
| 2 | `Load Active Clients` | Google Sheets → Read Rows | Pull all `active = TRUE` rows from `Clients` | Filter on `active` |
| 3 | `Loop Clients` | Split In Batches | One client at a time (keeps error isolation per client) | Batch size 1 |
| 4 | `Load Client Campaigns` | HTTP Request (Google Ads) | GAQL: `SELECT campaign.id, campaign.name FROM campaign WHERE campaign.status='ENABLED'` | uses client's `customer_id` |
| 5 | `Loop Campaigns` | Split In Batches | One campaign at a time | Batch size 1 |
| 6 | `Get Current Performance` | HTTP Request (Google Ads) | §4.1, current 14-day window | |
| 7 | `Get Baseline Performance` | HTTP Request (Google Ads) | §4.1, prior 14-day window | |
| 8 | `Get Search Terms` | HTTP Request (Google Ads) | §4.2 | |
| 9 | `Get Keyword Performance` | HTTP Request (Google Ads) | §4.3 | |
| 10 | `Get Ad Performance` | HTTP Request (Google Ads) | §4.4 | |
| 11 | `Get Change Events` | HTTP Request (Google Ads) | §4.5 | |
| 12 | `Get Google Recommendations` | HTTP Request (Google Ads) | §4.6 | |
| 13 | `Get GA4 Landing Page Data` | HTTP Request (GA4) | §4.7 — mark optional/skippable if client has no GA4 | Continue On Fail = true |
| 14 | `Merge All Reports` | Merge (Combine by position) | Assembles one object per campaign with all raw report arrays | |
| 15 | `Data Quality Gate` | Code | Validates completeness/staleness (§7) | outputs `is_valid`, `issues[]` |
| 16 | `IF: Data Valid?` | IF | Branch on `is_valid` | |
| 16a | `Alert: Data Issue` | Slack | Posts to client's channel with the missing/stale data | *(NO branch)* |
| 17 | `Calculate KPIs` | Code | §7.1 formulas | |
| 18 | `Detect Anomalies` | Code | §7.2 thresholds + volume gate | outputs `material_change_detected` |
| 19 | `IF: Material Change?` | IF | | |
| 19a | `Compile Report-Only Summary` | Code + Slack | Lightweight "nothing material this week" digest | *(NO branch)* |
| 20 | `Classify Search Terms` | Code | §7.3 rule-based pass | flags `needs_llm_review` |
| 21 | `Detect Cannibalization` | Code | §7.4 overlapping-query grouping | |
| 22 | `Correlate Changes to Anomaly Window` | Code | §7.5 — did a budget/keyword/tracking change coincide with the anomaly? | |
| 23 | `Build Evidence Pack` | Code | Assembles the JSON in §8 | |
| 24 | `LLM Strategist Call` | HTTP Request (Anthropic) or Anthropic node | System prompt = §9, user message = Evidence Pack | `max_tokens: 2000`, low temperature |
| 25 | `Validate LLM Output` | Code | Parse JSON, validate against schema §10; on failure, retry once with an error-correction follow-up message | |
| 26 | `Apply Policy Gate` | Code | §11 — tags each action with a tier and `approval_required` | |
| 27 | `Append to Ledger (pending)` | Google Sheets → Append Row | One row per recommended action | |
| 28 | `Send Approval Request` | Slack (Block Kit) | §12 — interactive Approve/Reject/Modify buttons | posts to client's `slack_channel_id` |
| 29 | `Approval Webhook` | Webhook (separate workflow) | Receives Slack interactivity payload | See §12.2 |
| 30 | `IF: Approved?` | IF | | |
| 30a | `Execute in Google Ads` | HTTP Request (Google Ads mutate) | §13 — only for the 2 mutate-capable action types (negatives, budget) | |
| 30b | `Log Rejection` | Google Sheets → Update Row | | |
| 31 | `Update Ledger (resolved)` | Google Sheets → Update Row | status, approver, timestamp | |
| 32 | `Client Digest` | Slack (or Google Docs API in v2) | Weekly summary: what was found, what was recommended, what was approved | |

Wrap nodes 6–13 with **Continue On Fail = true** and route failures to a shared `Error Handler` sub-workflow (Slack alert with the client/campaign/node that failed) so one client's API hiccup doesn't kill the whole sweep.

---

## 6. Client Strategy Layer Lookup (used inside Code nodes)

Every downstream Code node reads the current client row (already in the item's JSON from the Loop Clients batch) rather than hardcoding thresholds. This is what makes "$72 CPA" mean something — see §8's `client` block.

---

## 7. Deterministic Logic (Code nodes — JavaScript)

### 7.1 KPI Calculation

```javascript
// Input: items[0].json = { current: {...totals}, baseline: {...totals} }
function safeDiv(n, d) { return d > 0 ? n / d : null; }

function computeKpis(t) {
  return {
    impressions: t.impressions,
    clicks: t.clicks,
    cost: t.cost_micros / 1_000_000,
    conversions: t.conversions,
    ctr: safeDiv(t.clicks, t.impressions),
    cvr: safeDiv(t.conversions, t.clicks),
    cpc: safeDiv(t.cost_micros / 1_000_000, t.clicks),
    cpa: safeDiv(t.cost_micros / 1_000_000, t.conversions),
    roas: safeDiv(t.conversions_value, t.cost_micros / 1_000_000),
  };
}

function pctChange(cur, base) {
  if (cur == null || base == null || base === 0) return null;
  return (cur - base) / base;
}

const current = computeKpis(items[0].json.current);
const baseline = computeKpis(items[0].json.baseline);

const deltas = {
  ctr_pct_change: pctChange(current.ctr, baseline.ctr),
  cvr_pct_change: pctChange(current.cvr, baseline.cvr),
  cpc_pct_change: pctChange(current.cpc, baseline.cpc),
  cpa_pct_change: pctChange(current.cpa, baseline.cpa),
  roas_pct_change: pctChange(current.roas, baseline.roas),
};

return [{ json: { current_kpis: current, baseline_kpis: baseline, deltas } }];
```

### 7.2 Anomaly Detection (threshold + minimum-volume gate)

The volume gate matters more than the threshold — this is what stops the system from calling a CPA swing "material" off of three clicks.

```javascript
const MIN_CLICKS_FOR_CTR_SIGNAL = 30;
const MIN_CONVERSIONS_FOR_CPA_SIGNAL = 10;

const THRESHOLDS = {
  cpa_pct_change_bad: 0.20,    // CPA up 20%+
  cvr_pct_change_bad: -0.15,   // CVR down 15%+
  ctr_pct_change_bad: -0.15,
  roas_pct_change_bad: -0.15,
};

const { deltas } = items[0].json;
const c = items[0].json.current; // raw totals: clicks, conversions

const anomalies = [];
let volumeSufficient = true;

if (c.clicks >= MIN_CLICKS_FOR_CTR_SIGNAL) {
  if (deltas.ctr_pct_change !== null && deltas.ctr_pct_change <= THRESHOLDS.ctr_pct_change_bad) {
    anomalies.push({ metric: 'ctr', direction: 'down', pct_change: deltas.ctr_pct_change });
  }
} else {
  volumeSufficient = false;
}

if (c.conversions >= MIN_CONVERSIONS_FOR_CPA_SIGNAL) {
  if (deltas.cpa_pct_change !== null && deltas.cpa_pct_change >= THRESHOLDS.cpa_pct_change_bad) {
    anomalies.push({ metric: 'cpa', direction: 'up', pct_change: deltas.cpa_pct_change });
  }
  if (deltas.cvr_pct_change !== null && deltas.cvr_pct_change <= THRESHOLDS.cvr_pct_change_bad) {
    anomalies.push({ metric: 'cvr', direction: 'down', pct_change: deltas.cvr_pct_change });
  }
  if (deltas.roas_pct_change !== null && deltas.roas_pct_change <= THRESHOLDS.roas_pct_change_bad) {
    anomalies.push({ metric: 'roas', direction: 'down', pct_change: deltas.roas_pct_change });
  }
} else {
  volumeSufficient = false;
}

return [{
  json: {
    anomalies,
    material_change_detected: anomalies.length > 0,
    low_volume_flag: !volumeSufficient,
  }
}];
```

### 7.3 Search Term Classification (rule-based first pass)

```javascript
// negative_intent_dictionary comes from the client's Clients-sheet row
const negativeSignals = $('Load Active Clients').item.json.negative_keyword_dictionary
  .split(',').map(s => s.trim().toLowerCase());

const results = [];
for (const item of items) {
  const term = (item.json.search_term || '').toLowerCase();
  const clicks = item.json.clicks || 0;
  const conversions = item.json.conversions || 0;

  const hasNegativeSignal = negativeSignals.some(sig => term.includes(sig));

  let classification;
  if (conversions > 0) {
    classification = 'PROMOTE';
  } else if (hasNegativeSignal) {
    classification = 'NEGATIVE';
  } else if (clicks >= 10 && conversions === 0) {
    classification = 'NEGATIVE'; // spend at volume, zero conversions
  } else {
    classification = 'RESEARCH'; // ambiguous — could route to an LLM classification pass in v2
  }

  results.push({ json: { ...item.json, classification } });
}
return results;
```

### 7.4 Keyword Cannibalization Check

```javascript
// Groups keyword_view rows by normalized query text across campaigns/ad groups
const normalize = s => s.toLowerCase().replace(/[\[\]"+]/g, '').trim();

const groups = {};
for (const item of items) {
  const key = normalize(item.json.keyword_text);
  if (!groups[key]) groups[key] = [];
  groups[key].push({
    campaign_id: item.json.campaign_id,
    ad_group_id: item.json.ad_group_id,
    match_type: item.json.match_type,
    clicks: item.json.clicks,
  });
}

const cannibalization_flags = Object.entries(groups)
  .filter(([_, entries]) => new Set(entries.map(e => e.campaign_id)).size > 1)
  .map(([term, entries]) => ({ normalized_term: term, competing_entries: entries }));

return [{ json: { cannibalization_flags } }];
```

### 7.5 Change Correlation

```javascript
// Flags any account change_event within the anomaly window, so the Evidence
// Pack can distinguish "something changed" from "nothing changed."
const anomalyWindowStart = items[0].json.window_start; // ISO date
const changeEvents = items[0].json.change_events || [];

const relevantChanges = changeEvents.filter(ev =>
  new Date(ev.change_date_time) >= new Date(anomalyWindowStart)
);

return [{ json: {
  changes_in_window: relevantChanges.map(ev => ({
    date: ev.change_date_time,
    resource_type: ev.change_resource_type,
    operation: ev.resource_change_operation,
    changed_fields: ev.changed_fields,
    user: ev.user_email,
  }))
} }];
```

---

## 8. Evidence Pack (JSON Schema + Example)

This is the *only* thing the LLM sees — never raw report rows.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "EvidencePack",
  "type": "object",
  "required": ["client", "campaign", "performance", "changes", "search_terms", "anomalies"],
  "properties": {
    "client": {
      "type": "object",
      "required": ["client_id", "objective", "target_cpa", "monthly_budget"],
      "properties": {
        "client_id": { "type": "string" },
        "objective": { "type": "string" },
        "target_cpa": { "type": "number" },
        "target_roas": { "type": ["number", "null"] },
        "monthly_budget": { "type": "number" },
        "gross_margin_pct": { "type": ["number", "null"] }
      }
    },
    "campaign": {
      "type": "object",
      "required": ["id", "name", "status"],
      "properties": {
        "id": { "type": "string" },
        "name": { "type": "string" },
        "status": { "type": "string" }
      }
    },
    "performance": {
      "type": "object",
      "required": ["current", "baseline", "deltas"],
      "properties": {
        "current": { "$ref": "#/definitions/kpi_block" },
        "baseline": { "$ref": "#/definitions/kpi_block" },
        "deltas": { "type": "object" }
      }
    },
    "changes": { "type": "array", "items": { "type": "object" } },
    "search_terms": {
      "type": "object",
      "properties": {
        "new_low_intent_terms": { "type": "integer" },
        "new_converting_terms": { "type": "integer" },
        "potential_negatives": { "type": "integer" },
        "cannibalization_flags": { "type": "array", "items": { "type": "object" } }
      }
    },
    "competition": {
      "type": "object",
      "properties": {
        "impression_share_change": { "type": ["number", "null"] },
        "position_above_rate_change": { "type": ["number", "null"] }
      }
    },
    "anomalies": { "type": "array", "items": { "type": "object" } },
    "google_recommendations": { "type": "array", "items": { "type": "object" } }
  },
  "definitions": {
    "kpi_block": {
      "type": "object",
      "properties": {
        "impressions": { "type": "number" },
        "clicks": { "type": "number" },
        "cost": { "type": "number" },
        "conversions": { "type": "number" },
        "ctr": { "type": ["number", "null"] },
        "cvr": { "type": ["number", "null"] },
        "cpc": { "type": ["number", "null"] },
        "cpa": { "type": ["number", "null"] },
        "roas": { "type": ["number", "null"] }
      }
    }
  }
}
```

Example populated instance:

```json
{
  "client": {
    "client_id": "carpet-co-appleton",
    "objective": "qualified lead generation",
    "target_cpa": 65,
    "monthly_budget": 15000,
    "gross_margin_pct": 0.42
  },
  "campaign": { "id": "111222333", "name": "Commercial Carpet Cleaning", "status": "ENABLED" },
  "performance": {
    "current": { "impressions": 18421, "clicks": 1122, "cost": 7340, "conversions": 89, "ctr": 0.0609, "cvr": 0.0793, "cpc": 6.54, "cpa": 82.47, "roas": null },
    "baseline": { "impressions": 17902, "clicks": 1015, "cost": 5967, "conversions": 107, "ctr": 0.071, "cvr": 0.105, "cpc": 5.88, "cpa": 55.77, "roas": null },
    "deltas": { "ctr_pct_change": -0.142, "cvr_pct_change": -0.245, "cpc_pct_change": 0.112, "cpa_pct_change": 0.479 }
  },
  "changes": ["budget increased 23% 8 days ago"],
  "search_terms": { "new_low_intent_terms": 17, "new_converting_terms": 4, "potential_negatives": 12, "cannibalization_flags": [] },
  "competition": { "impression_share_change": -0.08, "position_above_rate_change": 0.14 },
  "anomalies": [
    { "metric": "cpa", "direction": "up", "pct_change": 0.479 },
    { "metric": "cvr", "direction": "down", "pct_change": -0.245 }
  ],
  "google_recommendations": []
}
```

---

## 9. LLM Strategist — System Prompt

Use this as the **system** message; the Evidence Pack JSON goes in as the **user** message on each call.

```
ROLE
You are the Senior Paid Search Strategist inside an automated Google Ads optimization
system used by a fractional Chief Marketing Officer. Your job is to interpret validated
performance data, diagnose the most likely causes of material performance changes,
distinguish symptoms from root causes, apply the SEM policy and client strategy below,
and produce recommendations a human executive can approve or reject. You do not perform
raw data collection or arithmetic, and you do not have unrestricted account access.

CORE PRINCIPLES
1. Business outcomes before platform metrics. Optimize toward the client's stated
   objective and economics (target CPA, ROAS, margin) — not CTR or clicks for their
   own sake.
2. Deterministic calculations are authoritative. Never recalculate or invent metrics.
   If required data is missing, say exactly what's missing and lower your confidence.
3. Diagnose before recommending. Never jump from "CPA increased" straight to "reduce
   budget." First determine whether the evidence points to keyword/query quality, ad
   relevance, landing-page performance, budget pressure, bidding behavior, competition,
   audience/device/geo mix, tracking, attribution, seasonality, insufficient sample
   size, or a deliberate account change.
4. Use the diagnostic framework:
   LOW IMPRESSIONS -> investigate keyword coverage, eligibility, targeting, bids, budget.
   GOOD IMPRESSIONS + LOW CTR -> investigate keyword relevance, ad copy, competitive
     positioning, SERP presence.
   GOOD CLICKS + LOW CVR -> investigate search intent, landing-page experience, offer
     alignment, conversion friction, tracking.
5. Treat search terms as a primary source of truth. Never recommend broad keyword
   expansion just because volume is high — prefer intent-aligned opportunities.
6. Respect historical context; don't overreact to short-term volatility the data
   doesn't support.
7. Separate FACTS (directly supported by the data), HYPOTHESES (plausible but
   unproven), and RECOMMENDATIONS (actions justified by the evidence). Never present
   a hypothesis as a fact.
8. Prefer reversible actions with measurable upside and limited downside.
9. Human approval is mandatory for material account changes — budget changes, bid
   strategy changes, restructuring, tracking/attribution changes, landing-page
   changes, substantial keyword expansion, and major ad messaging changes.

CONFIDENCE
HIGH = strong evidence from multiple independent signals.
MEDIUM = evidence supports the diagnosis but competing explanations remain plausible.
LOW = insufficient or conflicting evidence.
Never assign HIGH confidence off a single metric.

PRIORITY
CRITICAL = immediate risk to spend, tracking, or business performance.
HIGH = material likely impact with clear evidence.
MEDIUM = meaningful opportunity, not urgent.
LOW = useful observation or optimization.

RECOMMENDATION QUALITY
Be specific. "Reduce budget" is not acceptable. "Reduce Campaign X's daily budget by
10-15%, subject to approval, because current CPA is $82 against a $65 target while
CVR has fallen 25% following a 23% budget increase and search-term quality has
deteriorated; the change is reversible and should be reassessed next window" is.

OUTPUT
Return valid JSON only — no prose, no markdown fences, matching exactly this schema:

{
  "executive_summary": "",
  "severity": "critical|high|medium|low|none",
  "primary_problem": "",
  "facts": [],
  "likely_root_causes": [
    { "cause": "", "evidence": [], "confidence": "high|medium|low" }
  ],
  "alternative_explanations": [],
  "recommended_actions": [
    {
      "action_type": "add_negative_keywords|promote_keywords|adjust_budget|ad_test|investigate_landing_page",
      "action": "",
      "reason": "",
      "expected_impact": "high|medium|low",
      "risk": "high|medium|low",
      "confidence": "high|medium|low",
      "priority": "critical|high|medium|low",
      "reversible": true
    }
  ],
  "what_not_to_change": [],
  "measurement_plan": "",
  "additional_data_needed": []
}

FINAL RULE
You are rewarded for producing the fewest, best-supported actions likely to improve
the client's business outcome while minimizing unnecessary risk — not for producing
many recommendations.
```

Note the schema drops `approval_required` from the LLM's output on purpose — that field is not the model's call to make. It's assigned deterministically by the Policy Gate in §11, using `action_type` as the key.

---

## 10. LLM Output Schema (for the validation Code node)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "StrategistOutput",
  "type": "object",
  "required": ["executive_summary", "severity", "primary_problem", "recommended_actions"],
  "properties": {
    "executive_summary": { "type": "string" },
    "severity": { "enum": ["critical", "high", "medium", "low", "none"] },
    "primary_problem": { "type": "string" },
    "facts": { "type": "array", "items": { "type": "string" } },
    "likely_root_causes": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["cause", "confidence"],
        "properties": {
          "cause": { "type": "string" },
          "evidence": { "type": "array", "items": { "type": "string" } },
          "confidence": { "enum": ["high", "medium", "low"] }
        }
      }
    },
    "alternative_explanations": { "type": "array", "items": { "type": "string" } },
    "recommended_actions": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["action_type", "action", "reason", "priority", "confidence"],
        "properties": {
          "action_type": {
            "enum": ["add_negative_keywords", "promote_keywords", "adjust_budget", "ad_test", "investigate_landing_page"]
          },
          "action": { "type": "string" },
          "reason": { "type": "string" },
          "expected_impact": { "enum": ["high", "medium", "low"] },
          "risk": { "enum": ["high", "medium", "low"] },
          "confidence": { "enum": ["high", "medium", "low"] },
          "priority": { "enum": ["critical", "high", "medium", "low"] },
          "reversible": { "type": "boolean" }
        }
      }
    },
    "what_not_to_change": { "type": "array", "items": { "type": "string" } },
    "measurement_plan": { "type": "string" },
    "additional_data_needed": { "type": "array", "items": { "type": "string" } }
  }
}
```

Validation node logic:

```javascript
const Ajv = require('ajv'); // if the n8n instance allows npm modules in Code nodes;
                             // otherwise hand-roll the required-field checks below.
let parsed;
try {
  parsed = JSON.parse(items[0].json.llm_raw_text);
} catch (e) {
  return [{ json: { valid: false, error: 'not_valid_json', raw: items[0].json.llm_raw_text } }];
}

const requiredTopLevel = ['executive_summary', 'severity', 'primary_problem', 'recommended_actions'];
const missing = requiredTopLevel.filter(k => !(k in parsed));

if (missing.length > 0) {
  return [{ json: { valid: false, error: `missing_fields: ${missing.join(',')}`, raw: parsed } }];
}

return [{ json: { valid: true, ...parsed } }];
```

On `valid: false`, branch back to node 24 with a follow-up user message: *"Your previous response was not valid JSON matching the required schema. Return only the corrected JSON."* Retry once; if it fails twice, alert a human instead of guessing.

---

## 11. Policy / Risk Gate

Every `action_type` maps to a fixed tier — the LLM never decides its own approval requirement.

| action_type | Tier (v1) | Notes |
|---|---|---|
| `add_negative_keywords` | Tier 1 — Recommend | Mutate-capable in v1 once approved (§13.1) |
| `promote_keywords` | Tier 1 — Recommend | Manual execution in v1 (keyword/ad-group creation not yet automated) |
| `adjust_budget` | Tier 1 — Recommend | Mutate-capable in v1 once approved (§13.2) |
| `ad_test` | Tier 1 — Recommend | Manual execution — RSA asset creation is a v2 addition |
| `investigate_landing_page` | Tier 0 — Inform | Never auto-executes; surfaced to the client, no Google Ads mutate at all |

```javascript
const TIER_RULES = {
  add_negative_keywords: 'tier1_recommend',
  promote_keywords: 'tier1_recommend',
  adjust_budget: 'tier1_recommend',
  ad_test: 'tier1_recommend',
  investigate_landing_page: 'tier0_inform',
};

function classify(action) {
  const tier = TIER_RULES[action.action_type] || 'tier0_inform';
  return { ...action, policy_tier: tier, approval_required: tier !== 'tier2_auto' };
}

const output = items[0].json;
output.recommended_actions = output.recommended_actions.map(classify);
return [{ json: output }];
```

**Do not add a `tier2_auto` value until the Optimization Ledger has enough resolved rows per client, per action_type, with a documented acceptance and success rate** (see §15). That threshold, not a fixed calendar date, is what should unlock automation.

---

## 12. Approval Flow

### 12.1 Slack Block Kit message (one block per action)

```json
{
  "channel": "{{client_slack_channel_id}}",
  "blocks": [
    {
      "type": "section",
      "text": { "type": "mrkdwn", "text": "*CPA deterioration — Commercial Carpet Cleaning*\nCPA is $82.47 vs. $65 target (+48%). CVR down 24.5%. Likely cause: search-term quality decline following a budget increase." }
    },
    {
      "type": "section",
      "text": { "type": "mrkdwn", "text": ":one: *Add 12 negative keywords*\nEmployment-intent queries, $612 spend, 0 conversions.\nConfidence: HIGH | Risk: LOW" }
    },
    {
      "type": "actions",
      "block_id": "ledger_id_abc123",
      "elements": [
        { "type": "button", "text": { "type": "plain_text", "text": "Approve" }, "style": "primary", "action_id": "approve_action", "value": "abc123" },
        { "type": "button", "text": { "type": "plain_text", "text": "Reject" }, "style": "danger", "action_id": "reject_action", "value": "abc123" },
        { "type": "button", "text": { "type": "plain_text", "text": "Modify" }, "action_id": "modify_action", "value": "abc123" }
      ]
    }
  ]
}
```

Repeat the section+actions block pair per recommended action so each can be approved independently rather than as an all-or-nothing batch.

### 12.2 Approval Webhook (separate n8n workflow)

1. **Webhook node** receives Slack's interactivity POST.
2. **Code node** parses `payload.actions[0].action_id` and `.value` (the `ledger_id`).
3. **IF node** branches on `approve_action` / `reject_action` / `modify_action`.
4. Approve → triggers node 30a (§13 mutate) with the ledger row's `action_detail`.
5. Reject/Modify → updates the Ledger row and, for Modify, posts a follow-up Slack message asking for the edited parameters (e.g., a different budget %).
6. Respond to Slack's webhook within 3 seconds with a `200` and an updated Block Kit message (strike through the buttons, show "Approved by @name at [time]").

---

## 13. Execution (Google Ads Mutates — v1 scope only)

Only two action types mutate Google Ads directly in v1. Everything else stays manual/advisory.

### 13.1 Add Negative Keywords

```
POST https://googleads.googleapis.com/v23/customers/{{customer_id}}/campaignCriteria:mutate
{
  "operations": [
    {
      "create": {
        "campaign": "customers/{{customer_id}}/campaigns/{{campaign_id}}",
        "negative": true,
        "keyword": { "text": "carpet cleaning jobs", "matchType": "PHRASE" }
      }
    }
  ]
}
```

Batch all approved negatives for a campaign into one `operations` array rather than one call per keyword.

### 13.2 Adjust Budget

```
POST https://googleads.googleapis.com/v23/customers/{{customer_id}}/campaignBudgets:mutate
{
  "operations": [
    {
      "update": {
        "resourceName": "customers/{{customer_id}}/campaignBudgets/{{budget_id}}",
        "amountMicros": "45000000"
      },
      "updateMask": "amount_micros"
    }
  ]
}
```

After either mutate, immediately re-fetch the affected resource and write the before/after state into the Ledger row's `before_metrics` / a `change_confirmed` field — don't assume the mutate succeeded just because the HTTP call returned 200.

---

## 14. Data Quality Gate (node 15 detail)

```javascript
const issues = [];
const c = items[0].json.current;
const b = items[0].json.baseline;

if (!c || !b) issues.push('missing_performance_data');
if (c && c.impressions === 0 && c.clicks === 0) issues.push('zero_activity_current_period');
if (c && c.clicks > 0 && c.impressions === 0) issues.push('impossible_ratio_clicks_gt_impressions_denominator');

// staleness: flag if the API returned data more than 2 days old for the "current" window's end date
const today = new Date();
const dataEndDate = new Date(items[0].json.current_window_end);
const staleDays = Math.floor((today - dataEndDate) / (1000 * 60 * 60 * 24));
if (staleDays > 2) issues.push(`stale_data_${staleDays}_days`);

return [{ json: { is_valid: issues.length === 0, issues } }];
```

---

## 15. Optimization Ledger → Feedback Loop (v2)

Once v1 is running, add a second scheduled workflow (monthly) that:

1. Reads Ledger rows with `status = 'executed'` and `date_resolved` ≥ 30 days ago.
2. Re-pulls current performance for that campaign.
3. Compares against `before_metrics` and writes `after_metrics_30d` + `outcome`.
4. Aggregates by `client_id` + `action_type` into an acceptance-rate / success-rate table — this is the evidence you'd use to justify moving any `action_type` from Tier 1 to Tier 2 for a specific, well-performing client.

---

## 16. MVP Acceptance Criteria

Before calling v1 "done," it should be able to:

- [ ] Run unattended on the weekly schedule across ≥2 real client accounts without manual intervention
- [ ] Correctly skip/alert (not crash) when a client's GA4 or Google Ads call fails
- [ ] Produce an Evidence Pack that a human could read and reach the same diagnosis from
- [ ] Reject its own malformed LLM output at least once in testing and successfully retry
- [ ] Post a Slack approval request with working Approve/Reject buttons
- [ ] Execute an approved negative-keyword addition and confirm it landed in Google Ads
- [ ] Execute an approved budget change and confirm the new amount
- [ ] Log every recommendation — approved or not — into the Ledger with a full audit trail

## 17. Roadmap Beyond v1

| Version | Adds |
|---|---|
| v1 (this spec) | 3 diagnoses, 5 recommendation types, Sheets storage, Tier 1 approval only |
| v2 | Postgres ledger, feedback loop (§15), auction-insights report (async `AuctionInsightService`), RSA ad-copy generation, Tier 2 safe automation for proven action types |
| v3 | CRM close-loop revenue data, cost-per-qualified-lead as the optimization target, cross-client institutional-knowledge queries against the Ledger, competitor intelligence subsystem |

---

*Built on the UW-Madison SEM course materials (KPI alignment, the three-legged-stool keyword→ad→landing-page diagnostic framework, search-term analysis, conversion tracking) and the architecture review that scoped this system.*
