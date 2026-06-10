# Ramp Stack Warm Signal Monitor
### An n8n + OpenAI workflow that detects accounting firm staffing pressure and routes hot leads to AEs automatically — built in public for Ramp Stack's GTM team.

---

## What this does

Accounting firms under staffing pressure are the highest-intent prospects for [Ramp Stack](https://ramp.com/stack) — an AI operating system that automates the month-end close. But identifying *which* of Ramp's 4,500+ firm partners are actively struggling with capacity requires monitoring dozens of signals daily across trade publications, job boards, and industry news.

This workflow does that automatically.

Every weekday at 7am, it:

1. Fetches fresh articles from accounting trade publications and job board APIs
2. Runs each article through an OpenAI intent classifier that scores staffing pressure 1–10
3. Filters out low-signal noise (score < 6)
4. Enriches high-signal firms with Apollo.io company data
5. Fires a rich Slack alert to the AE channel with a direct action recommendation
6. Upserts a HubSpot company record tagged with signal score, type, and date

The result: AEs start every morning with a prioritized list of firms to call — no manual research required.

---

## Workflow architecture

```
Schedule Trigger (7am weekdays)
        │
        ▼
RSS + News Fetch
(Accounting Today, CPA Practice Advisor, Job Boards)
        │
        ▼
OpenAI Intent Classifier
(GPT-4o-mini · scores staffing pressure 1–10)
        │
        ▼
IF Filter (score >= 6?)
    │           │
   YES          NO
    │           │
    ▼        Discard
Apollo Enrichment
(firm size, revenue, AE owner, LinkedIn)
        │
   ┌────┴────┐
   ▼         ▼
Slack     HubSpot
Alert     Upsert
```

---

## Stack

| Layer | Tool |
|---|---|
| Workflow orchestration | [n8n](https://n8n.io) |
| Intent classification | OpenAI GPT-4o-mini (native n8n node) |
| Firm enrichment | [Apollo.io](https://apollo.io) API |
| AE notification | Slack (Block Kit) |
| CRM logging | HubSpot |
| Signal sources | Accounting Today RSS, CPA Practice Advisor RSS, Adzuna Jobs API |

---

## Prerequisites

- n8n instance (cloud or self-hosted via Docker)
- OpenAI credential connected in n8n (the built-in node credential works — no separate API key needed)
- Apollo.io account + API key (toggle **Master API Key** on when generating)
- Slack bot token with scopes: `chat:write`, `channels:read`
- HubSpot private app token (Settings → Integrations → Private Apps)

---

## Node-by-node setup

### 1. Schedule Trigger
- **Mode:** Cron
- **Expression:** `0 7 * * 1-5` (weekdays at 7am)
- **Why:** Alerts land in Slack right when AEs start their day, not buried overnight

---

### 2. RSS + News Fetch

Add an **RSS Feed** node for each source:

| Source | Feed URL |
|---|---|
| Accounting Today | `https://www.accountingtoday.com/feed` |
| CPA Practice Advisor | `https://feeds.feedburner.com/cpapracticeadvisor` |
| Journal of Accountancy | `https://www.journalofaccountancy.com/rss` |

Add an **HTTP Request** node for job postings:
- **URL:** Adzuna Jobs API scoped to `("accounting firm" OR "CPA firm") AND ("hiring" OR "accountant")` — last 24 hours
- Merge all sources into one array using a **Merge** node (Append mode)
- Add a **Filter** node to drop anything older than 48 hours using `{{ $json.pubDate }}`

---

### 3. OpenAI Intent Classifier

**Node:** OpenAI (native n8n node — use your connected credential)  
**Model:** `gpt-4o-mini`  
**Max Tokens:** `300`  
**Temperature:** `0`  
**Response Format:** `JSON Object`

#### System prompt

```
You are a GTM intelligence analyst for Ramp Stack, an AI operating system built specifically for accounting firms.

Your job is to read news articles, job postings, and industry content and determine whether the content signals that a named accounting firm is experiencing staffing pressure — defined as any condition where the firm cannot fully serve its current or potential client base due to talent constraints.

Staffing pressure signals include:
- Active hiring for multiple accounting/bookkeeping/CPA roles
- Mentions of staff departures, attrition, or difficulty retaining talent
- Firm turning away clients or pausing business development
- Merger or acquisition driven by capacity needs (not just growth)
- Managing partner commentary about workload, burnout, or pipeline problems
- Accounting degree pipeline concerns applied to a specific firm's situation

Do NOT score highly for:
- General industry articles with no named firm
- Firms hiring for non-accounting roles only (IT, marketing, admin)
- Articles that mention a firm positively growing with no capacity strain
- Press releases about awards, rankings, or certifications

Scoring guide:
9-10 → Named firm, explicit staffing crisis, direct quote from leadership or multiple open roles posted
7-8  → Named firm, clear capacity strain implied, at least one concrete data point
5-6  → Named firm mentioned but signal is inferential
3-4  → No specific firm named, general industry signal only
1-2  → Tangentially related content, no actionable signal

You must respond with ONLY a valid JSON object. No explanation, no markdown.
```

#### User prompt

```
Analyze the following content and return your assessment.

Title: {{ $json.title }}
Publication: {{ $json.source }}
Date: {{ $json.pubDate }}
URL: {{ $json.link }}
Content: {{ $json.description }}

Respond with exactly this JSON structure:
{
  "score": <integer 1-10>,
  "firm": "<named firm or 'unknown'>",
  "signal_type": "<hiring_surge | staff_shortage | merger_capacity | client_overflow | partner_commentary | general_industry>",
  "confidence": "<high | medium | low>",
  "rationale": "<one sentence, max 20 words>",
  "ae_action": "<call_today | add_to_sequence | monitor | discard>"
}
```

---

### 4. Code node — parse OpenAI output

The OpenAI node returns content nested at `output[0].content[0].text`. This Code node extracts and normalizes it:

```javascript
const input = $input.first().json;

const raw = input.output[0].content[0].text.trim();

const parsed = JSON.parse(raw);

return [{
  json: {
    score: parseInt(parsed.score),
    firm_name: parsed.firm,
    reason: parsed.rationale,
    signal_type: parsed.signal_type || 'general_industry',
    confidence: parsed.confidence || 'medium',
    ae_action: parsed.ae_action || (
      parsed.score >= 8 ? 'call_today' :
      parsed.score >= 6 ? 'add_to_sequence' : 'discard'
    ),
    source_title: $input.first().json.title,
    source_url: $input.first().json.link
  }
}];
```

> **Note:** The `ae_action` fallback computes the recommended action from the score if the model skips the field. The IF node downstream always receives a clean integer `score`.

---

### 5. IF Filter

- **Value 1:** `{{ $json.score }}`
- **Operation:** `is greater than or equal to`
- **Value 2:** `6`

True branch → Apollo enrichment  
False branch → No Operation (discard)

Start at 6 and tune after the first week. If AEs report too much noise, raise to 7. If the channel is too quiet, lower to 5.

---

### 6. Apollo Enrichment

**HTTP Request node:**

```
POST https://api.apollo.io/api/v1/organizations/search
Header: x-api-key: YOUR_APOLLO_KEY
```

```json
{
  "q_organization_name": "{{ $json.firm_name }}",
  "organization_industries": ["accounting"],
  "per_page": 1
}
```

Follow with a Code node to extract relevant fields:

```javascript
const org = $input.first().json.organizations?.[0];
const signal = $input.first().json;

return [{
  json: {
    firm_name: org?.name || signal.firm_name,
    headcount: org?.estimated_num_employees,
    revenue_range: org?.annual_revenue_printed,
    website: org?.website_url,
    linkedin: org?.linkedin_url,
    city: org?.city,
    state: org?.state,
    signal_score: signal.score,
    signal_type: signal.signal_type,
    confidence: signal.confidence,
    reason: signal.reason,
    ae_action: signal.ae_action,
    source_title: signal.source_title,
    source_url: signal.source_url
  }
}];
```

---

### 7a. Slack Alert

**Node:** Slack (native n8n node)  
**Channel:** `#stack-signals`  
**Message type:** Blocks

```json
[
  {
    "type": "section",
    "text": {
      "type": "mrkdwn",
      "text": "*📶 Stack signal — score {{ $json.signal_score }}/10*\n*Firm:* {{ $json.firm_name }}\n*Signal:* {{ $json.reason }}"
    }
  },
  {
    "type": "section",
    "fields": [
      { "type": "mrkdwn", "text": "*Headcount:* {{ $json.headcount || 'unknown' }}" },
      { "type": "mrkdwn", "text": "*Location:* {{ $json.city }}, {{ $json.state }}" },
      { "type": "mrkdwn", "text": "*Signal type:* {{ $json.signal_type }}" },
      { "type": "mrkdwn", "text": "*Confidence:* {{ $json.confidence }}" },
      { "type": "mrkdwn", "text": "*Recommended action:* {{ $json.ae_action }}" },
      { "type": "mrkdwn", "text": "*Revenue:* {{ $json.revenue_range || 'unknown' }}" }
    ]
  },
  {
    "type": "actions",
    "elements": [
      { "type": "button", "text": { "type": "plain_text", "text": "View source" }, "url": "{{ $json.source_url }}" },
      { "type": "button", "text": { "type": "plain_text", "text": "LinkedIn" }, "url": "{{ $json.linkedin }}" }
    ]
  }
]
```

---

### 7b. HubSpot Upsert

**HTTP Request node:**

```
POST https://api.hubapi.com/crm/v3/objects/companies/upsert
Header: Authorization: Bearer YOUR_HUBSPOT_TOKEN
```

```json
{
  "properties": {
    "name": "{{ $json.firm_name }}",
    "website": "{{ $json.website }}",
    "numberofemployees": "{{ $json.headcount }}",
    "stack_signal_score": "{{ $json.signal_score }}",
    "stack_signal_type": "{{ $json.signal_type }}",
    "stack_signal_reason": "{{ $json.reason }}",
    "stack_signal_date": "{{ $now }}"
  },
  "idProperty": "name"
}
```

Build a HubSpot saved view filtered by `stack_signal_score >= 6` and `stack_signal_date within 7 days` — this becomes your AEs' daily Stack prospecting list.

---

## Testing checklist

Before activating, run the workflow manually and verify each step:

- [ ] RSS nodes return articles with `title`, `description`, `pubDate`, `link` fields
- [ ] Filter node drops articles older than 48 hours
- [ ] OpenAI node returns a JSON string at `output[0].content[0].text`
- [ ] Code node produces a clean integer `score` (not `"8"` in quotes)
- [ ] IF node routes score >= 6 to the true branch
- [ ] Apollo returns a firm match for at least one test article
- [ ] Slack message renders correctly (use [Block Kit Builder](https://app.slack.com/block-kit-builder) to preview)
- [ ] HubSpot company record is created with all five `stack_signal_*` properties

---

## Tuning guide

| Parameter | Default | Raise if... | Lower if... |
|---|---|---|---|
| IF score threshold | 6 | AEs say too many weak leads | Channel is too quiet |
| Max tokens | 300 | Rationale gets cut off | Costs need trimming |
| Temperature | 0 | You want more varied output | Classifier is inconsistent |
| Cron schedule | Weekdays 7am | You need real-time signals | Too many daily executions |

---

## Known limitations

- **OpenAI free tier caps:** 200 requests/day. For batches above ~150 articles, upgrade to pay-as-you-go ($5 minimum — costs roughly $0.10–0.20/day at GPT-4o-mini pricing).
- **Apollo unmatched firms:** If `firm_name` comes back as `"unknown"` from the classifier, Apollo will return no results. The workflow still fires the Slack alert but flags it as unverified.
- **RSS freshness:** Most accounting trade feeds update once or twice daily. Job board API freshness depends on your Adzuna plan.

---

## Roadmap

- [ ] Add LinkedIn Sales Navigator signal source via RapidAPI
- [ ] Personalized outreach email generator (workflow 2) — feeds directly from this pipeline
- [ ] Competitive intelligence monitor for Karbon, Botkeeper, Canopy
- [ ] Weekly digest email summarizing all signals and AE actions taken

---

## Context

This workflow was built as part of a GTM automation project analyzing Ramp Stack's product launch — studying their ICP, value proposition, and go-to-market motion to identify where AI workflows could drive the most pipeline ROI.

The core insight: Ramp has 4,500 warm firm relationships but no visible mechanism for prioritizing which ones to activate first. This workflow closes that gap.

---

## Contributing

If you use this workflow, adapt it for a different vertical, or improve the classifier prompt — open a PR or share what you built. The prompt engineering and node configuration are the most transferable parts; the signal sources and enrichment layer can be swapped for any B2B GTM context.

---

*Built in public. Inspired by the gap between warm relationships and activated pipeline.*
