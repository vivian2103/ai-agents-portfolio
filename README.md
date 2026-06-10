# n8n × Claude Workflows
### A growing collection of AI-powered automation workflows built with n8n and Claude — covering GTM intelligence, marketing automation, and sales pipeline acceleration.

---

## What this repo is

This is my public library of production-ready n8n workflows. Every workflow here was built to solve a real business problem — not as a demo, but as infrastructure I actually run or built for clients.

The through-line across all of them: **n8n as the orchestration layer, Claude (or OpenAI) as the reasoning layer.** Most automation tools can move data between apps. These workflows make decisions about that data — classifying intent, generating personalized output, prioritizing leads, and routing actions — before anything gets sent anywhere.

I build in public. Every workflow ships with full node-by-node documentation, the exact prompts used, and honest notes on limitations and edge cases.

---

## Workflow index

### 🎯 GTM & sales intelligence

| Workflow | What it does | Stack |
|---|---|---|
| [Ramp Stack Warm Signal Monitor](./ramp-stack-signal-monitor/) | Monitors accounting trade press and job boards daily, classifies staffing pressure signals with AI, routes hot leads to AEs via Slack and HubSpot | n8n · OpenAI · Apollo · Slack · HubSpot |
| *More coming* | Personalized outreach generator for warm firm relationships | n8n · Claude · HubSpot |
| *More coming* | Competitive intelligence monitor (Karbon, Botkeeper, Canopy) | n8n · Claude · Slack |

### 📣 Content & marketing automation

| Workflow | What it does | Stack |
|---|---|---|
| *Coming soon* | Benchmark content repurposing engine — turns product data into LinkedIn posts, email snippets, and trade pitches in parallel | n8n · Claude · Buffer |
| *Coming soon* | LinkedIn post performance analyzer — classifies what's working and generates follow-up content variants | n8n · Claude · LinkedIn API |

### 🔁 Operations & enrichment

| Workflow | What it does | Stack |
|---|---|---|
| *Coming soon* | Meeting prep enrichment — enriches calendar invites with Apollo + LinkedIn data before every sales call | n8n · Apollo · Google Calendar |
| *Coming soon* | CRM hygiene agent — detects stale records, fills missing fields, flags ownership gaps | n8n · Claude · HubSpot |

---

## How each workflow is documented

Every folder in this repo follows the same structure:

```
workflow-name/
├── README.md          # Full node-by-node setup guide
├── workflow.json      # Importable n8n workflow file
└── prompts/
    ├── system.txt     # System prompt used in the AI node
    └── user.txt       # User prompt template with n8n expressions
```

The `workflow.json` can be imported directly into any n8n instance via **Settings → Import workflow**. All credentials are placeholder strings — replace them with your own before activating.

---

## Design principles

**Prompts are first-class artifacts.** The AI prompt is the most important part of any workflow in this repo. Every prompt ships with inline comments explaining why each instruction exists — not just what it says.

**Workflows fail gracefully.** Every Code node includes error handling that logs malformed AI responses without crashing the batch. A single bad article or unparseable response should never stop 50 others from processing.

**Outputs drive action, not just information.** Every workflow ends with a recommended next action — `call_today`, `add_to_sequence`, `monitor` — computed by the AI, not hardcoded by a rule. The human reviews, not decides.

**Documentation matches what actually works.** Notes on rate limits, API quirks, and edge cases are included because I hit them building these. Nothing here is theoretical.

---

## Tech stack overview

| Tool | Role |
|---|---|
| [n8n](https://n8n.io) | Workflow orchestration — triggers, routing, API calls, data transformation |
| [Claude](https://claude.ai) (Anthropic) | Long-form reasoning, structured classification, content generation |
| [OpenAI](https://openai.com) | Fast classification tasks, JSON-mode output, cost-sensitive batch jobs |
| [Apollo.io](https://apollo.io) | B2B firm and contact enrichment |
| [HubSpot](https://hubspot.com) | CRM record creation, contact and company upserts |
| [Slack](https://slack.com) | AE and team alerts via Block Kit |

---

## Background

I'm a marketing and GTM automation builder studying business at UGA Terry College of Business. These workflows come out of real client and consulting projects — analyzing product launches, building outbound pipelines, and automating the manual research work that slows GTM teams down.

If you're a founder, AE, or GTM operator who wants to adapt any of these for your own stack, everything here is open to fork, modify, and build on.

---

## Stay updated

New workflows drop as I build them. Watch the repo to get notified, or connect on [LinkedIn](#) where I write about what I'm building and why.

---

*Built in public. All prompts included. No fluff.*
