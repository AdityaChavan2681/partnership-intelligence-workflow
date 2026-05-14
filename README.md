# Partnership Intelligence Pipeline

Brand prospect assessment, sponsor-fit scoring, and review-first outreach preparation for sports partnership teams.

> Status: Active development. This repository currently documents the project architecture, workflow behavior, screenshots, and implementation notes. The n8n workflow export, credentials, API keys, database connections, and environment-specific configuration are not included yet.

This project uses n8n to evaluate companies as potential sponsors, partners, vendors, media partners, or activation partners for a target sports market. The current workflow is tuned for the pickleball ecosystem, but the intake fields and scoring model are designed to support other sports, events, venues, leagues, and commercial partnership categories.

The system is designed to answer:

```text
Given a target sport, league, event, venue, or sports property and an outside company,
is this company a credible partnership prospect, and what outreach angle should be reviewed?
```

---

## Repository Scope

This repository is currently README-only while the workflow is being developed and tested. It documents the system design, current workflow behavior, integrations, operating lessons, and portfolio screenshots before the n8n export and setup assets are prepared for release.

The working n8n workflow, local credentials, API keys, Zapier setup, MongoDB connection details, Airtable configuration, SMTP credentials, and temporary Cloudflare Tunnel URLs are intentionally excluded.

Local testing and debugging may use PowerShell commands for n8n API calls, execution-output exports, and webhook/tunnel checks.

---

## Overview

Partnership teams often evaluate prospects from scattered public signals: homepages, product pages, partner pages, athlete pages, event pages, blog posts, news pages, media pages, and contact pages. Those signals are useful, but they are noisy and hard to compare manually.

This pipeline converts public-web evidence into structured sponsor-prospect intelligence that can be queued, scored, reviewed, retried, stored, and turned into outreach-ready drafts.

The workflow combines:

- Zapier-compatible webhook intake
- n8n workflow orchestration
- MongoDB-backed queueing, dedupe, retry state, evidence storage, and run status
- Optional Apollo discovery when credits are available
- Optional Clay enrichment handoff
- Public website discovery and text extraction
- Gemma/Gemini-compatible model nodes for company context and page classification
- Deterministic scoring, normalization, and quality gates
- Short-term and long-term partnership potential scoring
- Airtable review outputs for assessments, rankings, contacts, and outreach
- Review-first outreach preparation with automatic owner-review email notifications
- Manual outreach tracking and sender-domain readiness fields

---

## Architecture

```text
Zapier or HTTP webhook trigger
→ n8n pipeline run webhook
→ pipeline run lock/status check
→ optional Apollo prospect discovery
   ├─ if Apollo succeeds: normalize and queue discovered companies in MongoDB
   ├─ if Apollo has no credits or fails: continue from the MongoDB queue
   └─ if Clay handoff is configured: optionally send enrichment payloads
→ select queued, retryable, or provider-failed assessment jobs from MongoDB
→ dedupe selected jobs by company/root domain
→ process selected companies sequentially through the assessment workflow
→ fetch homepage with browser-like headers
→ discover internal links and fallback probe URLs
→ filter and classify relevant website pages
→ fetch selected pages and validate HTTP/page quality
→ extract clean page text and preserve evidence snippets
→ build multi-page company context input
→ extract company-level context
→ attach company context to page-level records
→ split pages into classification batches
→ classify page-level partnership signals with model-backed extractor nodes
→ normalize and validate AI output
→ apply fallback handling for weak, blocked, or invalid pages
→ score page-level sponsor-fit evidence
→ store page-level evidence in MongoDB
→ aggregate brand-level assessment
→ compare against previous assessment state
→ write company assessment, ranking, contacts, and outreach review outputs
→ generate company-context outreach draft
→ send owner-review email through SMTP/Gmail when enabled
→ stage manual outreach status, sender-domain readiness, and contact-role targets
→ update MongoDB job status and pipeline run status
```

---

## Workflow State

Current exported workflow file:

```text
partnership-intelligence-workflow-revamped.json
```

Workflow snapshot:

- 114 n8n nodes after the latest outreach-email update
- Reduced from 119 nodes in the previous export before simplification
- Removed 7 wait nodes while preserving sequential company processing and extractor retry control
- Zapier-compatible webhook path: `partnership-pipeline-run`
- MongoDB queue collection: `brand_assessment_jobs`
- Page evidence collection: `brand_page_assessments`
- Company assessment collection: `brand_assessments`
- Outreach draft collection: `outreach_drafts`
- Pipeline status collections: `pipeline_run_status`, `pipeline_run_events`
- Current primary model routing: `models/gemma-4-26b-a4b-it`
- Extractor retries: 2 tries per extractor node
- Owner-review email node added for SMTP/Gmail notification when credentials are configured

The main assessment still processes companies sequentially. `max_companies_per_run` controls how many queued companies can be selected for a run, with the workflow capped at 5.

---

## Webhook Run Input

Zapier currently sends fields like this:

```json
{
  "search_keywords": "pickleball equipment",
  "search_location": "United States",
  "search_page_size": 5,
  "search_page_count": 1,
  "prospect_group": "brand_partner",
  "target_market_name": "Pickleball",
  "target_org_type": "sport",
  "target_market_description": "pickleball / sports / events",
  "prospect_type": "brand",
  "partnership_goal": "sponsorship",
  "partnership_horizon": "long_term",
  "preferred_partner_types": "equipment, media, facility, events",
  "target_regions": "United States",
  "minimum_company_size": 10,
  "maximum_company_size": 500,
  "revenue_priority": "medium",
  "growth_priority": "high",
  "local_activation_priority": "medium",
  "media_reach_priority": "high",
  "event_sponsorship_priority": "high",
  "product_fit_priority": "high",
  "contact_discovery_required": false,
  "max_companies_per_run": 1,
  "outreach_delivery_platform": "review_only",
  "sender_domain": "testdomain.com",
  "dkim_selector": "default",
  "manual_sender_email": "aditya@test.com"
}
```

Notes:

- `search_page_count` controls Apollo discovery pagination. It does not control website crawl depth.
- Website crawl limits are controlled by runtime config, including `crawl_config.max_pages_total`.
- `search_page_size` controls the Apollo page size when Apollo discovery is enabled.
- `max_companies_per_run` controls how many queued companies can be processed sequentially in one run.
- Outreach delivery fields are used for draft readiness and review tracking; prospect-facing sends are not automatic.
- During testing, `max_companies_per_run: 1` is recommended to avoid burning model quota while debugging.

---

## Data Flow

```text
Webhook request
→ pipeline run context
→ optional Apollo discovery
→ optional Clay enrichment handoff
→ MongoDB queue insert/update and dedupe
→ selected assessment jobs
→ public website discovery
→ page fetch and text extraction
→ company-context extraction
→ page-level partnership signal classification
→ deterministic scoring and quality gates
→ MongoDB evidence/history storage
→ Airtable review outputs
→ outreach draft and owner-review notification
```

Apollo is useful for discovery, but it is not required for assessment execution. If Apollo credits are unavailable, the workflow can still assess companies already present in MongoDB.

MongoDB is the operational store. Airtable is the review layer.

---

## Brand Assessment Workflow

The Brand Prospect Assessor evaluates an outside company as a potential sponsor or partner for a target sports market.

Starting from a company name and root domain, the workflow discovers and evaluates relevant pages such as:

- Homepage
- About pages
- Product and category pages
- Partner, sponsorship, affiliate, ambassador, or collaboration pages
- Blog, news, media, and press pages
- Event, athlete, club, facility, or tournament pages
- Contact pages as supporting context only

Each page is classified and scored individually. The workflow then aggregates the strongest page-level evidence into one company-level sponsor-fit assessment.

Before page-level classification, the workflow builds a multi-page company context record. This helps identify what the analyzed organization primarily does while keeping final scoring grounded in each page's own evidence.

---

## Scoring Basis

Scores are based on public-web evidence and support both short-term activation opportunities and long-term partnership evaluation.

The workflow rewards signals such as:

- sport/category relevance
- sponsor fit
- product or service alignment
- partnership language
- event/community activation potential
- evidence breadth
- source quality

The scoring logic combines page-level evidence scores with brand-level aggregation and quality gates. AI is used to classify and summarize evidence, while deterministic workflow logic handles scoring, caps, retry status, timeline potential, and validation.

The workflow also produces separate short-term and long-term partnership potential fields. Short-term potential reflects immediate campaign, event, media, product, venue, or activation signals. Long-term potential reflects durable sponsor fit, partnership language, evidence breadth, high-fit pages, and source quality.

---

## Review-First Outreach Workflow

The outreach layer prepares high-quality drafts and review records before any prospect-facing send step.

A completed publishable assessment can generate:

- a company-context outreach draft
- a suggested subject line and email body
- evidence used for personalization
- recommended contact roles
- manual outreach status fields
- sender-domain readiness fields for SPF, DKIM, and DMARC
- an automatic owner-review email through SMTP/Gmail when credentials are configured

The owner-review email can send automatically to the operator. Prospect-facing sends are intentionally gated behind human review, verified contact data, sender-domain readiness, and future platform approval rules.

Smartlead, Instantly, or similar outbound platforms remain planned as optional downstream delivery layers after human approval. The current workflow is review-first rather than bulk-send-first.

---

## Main Outputs

The workflow produces six review layers:

- Assessment: score, conclusion, recommended action, category, business model, and offer type
- Timeline: short-term partnership potential, long-term partnership potential, timeline recommendation, and rationale
- Evidence: source URLs, top evidence pages, evidence summaries, page counts, and signal flags
- Reliability: quality status, retry flag, provider-error count, fallback count, blocked-page count, and evidence strength
- Ranking: rank tier, reliability level, strategic fit, commercial fit, activation fit, and ranking rationale
- Outreach: draft email, owner-review notification, manual outreach status, sender-domain readiness, and contact-role targets

---

## Reliability Model

The workflow is designed to avoid publishing weak or failed assessments as successful partner recommendations.

Reliability behavior:

- Apollo duplicate-only runs can continue into existing queued or retryable MongoDB jobs.
- Apollo provider errors or credit limits do not block existing MongoDB queue assessment.
- Website evidence can override stale or incomplete enrichment metadata.
- Blocked, empty, failed, and soft-404 pages produce fallback records instead of disappearing silently.
- Suspicious or hijacked-looking domains can be rejected deterministically before wasting model calls.
- Model/provider failures are marked as `failed_provider` or `needs_retry` instead of being published as completed assessments.
- Quality gates prevent provider-error assessments from being treated as review-ready results.
- Failed-provider assessments are blocked from outreach until a publishable assessment exists.
- Reruns can suppress unchanged Airtable assessment/ranking updates while still allowing publishable outreach draft generation.

---

## Runtime and Model Notes

Model choice matters because each company can create several company-context and page-classification calls.

Operating notes:

- The workflow currently routes core model calls through `models/gemma-4-26b-a4b-it` in the exported file.
- `gemini-3.1-flash-lite` previously returned frequent service-unavailable errors, but later tests showed it can recover and may be faster for page classification when available.
- `gemma-4-26b-a4b-it` was more stable in earlier runs and useful for larger text inputs, but it also produced occasional provider-side 500 errors.
- Gemma 3 variants may be worth testing later for higher RPM/RPD, but their 15K TPM limit means page text may need tighter trimming.
- RPM, TPM, and RPD are the practical throughput limits; sampling settings are kept conservative for consistent structured extraction.

---

## MongoDB Collections

MongoDB stores operational state, evidence, outreach drafts, and historical assessment records.

Collections include:

- `brand_assessment_jobs`: lightweight queue, dedupe state, job status, retry status
- `brand_page_assessments`: page-level evidence and page scores
- `brand_assessments`: company-level assessment history
- `outreach_drafts`: outreach drafts, owner-review email payloads, manual outreach status, sender-domain readiness
- `config`: runtime config, taxonomy, and crawl settings
- `pipeline_run_status`: run lock and current run status
- `pipeline_run_events`: skip/no-work/status events

The queue is intentionally lightweight. Rich Apollo, Clay, scoring, ranking, and outreach fields are preserved in assessment, enrichment, ranking, and review outputs rather than being required as initial queue input.

---

## Airtable Review Tables

Airtable is used for human review, filtering, prioritization, and approval.

Review outputs use:

- `company_enrichment`
- `brand_assessments`
- `partnership_prospect_rankings`
- `prospect_contacts`
- `outreach_review_queue`

The `brand_assessments` table stores company-level sponsor-fit assessments. `partnership_prospect_rankings` stores prioritization records with score, fit, evidence strength, reliability level, and ranking rationale. `prospect_contacts` stores recommended contact-role placeholders and future person-level enrichment records. `outreach_review_queue` stores review-ready outreach drafts, manual outreach status, sender-domain readiness, and follow-up tracking fields.

---

## Integrations

### Zapier

Zapier is the current trigger layer. It sends the run request to the n8n webhook URL and supplies search, targeting, scoring-priority, run-size, and outreach-readiness fields.

### Cloudflare Tunnel

Cloudflare Tunnel is used during development to expose the local n8n webhook to external tools such as Zapier without deploying the workflow to a public server. Tunnel URLs are temporary development endpoints and are not included in the repository.

### Apollo

Apollo is optional discovery. It can supply prospect companies, domains, industry metadata, location data, employee estimates, LinkedIn URLs, and keyword signals. If Apollo credits are exhausted, the assessment pipeline can continue from MongoDB queue records.

### Clay

Clay is an optional enrichment/reference-data handoff. n8n can send company payloads to Clay when enrichment is needed, while the main assessment continues without requiring a Clay callback.

### MongoDB

MongoDB stores queue state, dedupe state, retry status, page-level evidence, company-level assessment history, pipeline run status, and outreach draft records.

### Airtable

Airtable stores review-ready outputs for sorting, filtering, prioritization, and approval.

### SMTP/Gmail

SMTP/Gmail can send automatic owner-review notifications for generated outreach drafts. These emails are sent to the operator for review and do not send prospect-facing outreach.

### Outreach Delivery Tools

Smartlead, Instantly, or similar outbound platforms are planned as optional downstream delivery layers after human review. Prospect-facing sends are intentionally gated behind approval, verified contact data, and sender-domain readiness.

---

## Verified Output Example

The latest exported test run completed successfully and produced a publishable assessment, ranking output, outreach draft, Airtable review row, contact-role records, and an owner-review email.

Verified execution summary:

```text
Execution ID: 1059
Status: success
Runtime: 50 seconds
Top-level errors: 0
Node errors: 0
Company: Selkirk Sport - We Are Pickleball
Root domain: https://selkirk.com
```

Company assessment output:

```json
{
  "analyzed_organization": "Selkirk Sport - We Are Pickleball",
  "root_domain": "https://selkirk.com",
  "company_category": "pickleball equipment",
  "brand_fit_score": 100,
  "recommended_action": "prioritize outreach",
  "assessment_quality_status": "complete",
  "short_term_partnership_potential": "medium",
  "short_term_partnership_potential_score": 73,
  "long_term_partnership_potential": "high",
  "long_term_partnership_potential_score": 100,
  "partnership_timeline_recommendation": "long-term partnership",
  "outreach_delivery_platform": "review_only",
  "sender_domain": "testdomain.com",
  "manual_sender_email": "aditya@test.com"
}
```

Ranking output:

```json
{
  "prospect_name": "Selkirk Sport - We Are Pickleball",
  "root_domain": "https://selkirk.com",
  "company_category": "pickleball equipment",
  "brand_fit_score": 100,
  "rank_tier": "A - prioritize",
  "recommended_action": "prioritize outreach",
  "reliability_level": "high confidence",
  "evidence_strength": "strong"
}
```

Outreach review output:

```json
{
  "analyzed_organization": "Selkirk Sport - We Are Pickleball",
  "root_domain": "https://selkirk.com",
  "brand_fit_score": 100,
  "recommended_action": "prioritize outreach",
  "outreach_delivery_platform": "review_only",
  "send_step_status": "review_only_no_send_platform",
  "sender_domain": "testdomain.com",
  "spf_status": "not_checked",
  "dkim_status": "not_checked",
  "dmarc_status": "not_checked",
  "owner_review_email_to": "chavanaditya0000@gmail.com",
  "owner_review_email_subject": "Review outreach draft: Selkirk Sport - We Are Pickleball (score 100)"
}
```

The outreach branch creates review-ready email drafts, owner-review email notifications, manual outreach status fields, and contact-role placeholders before any prospect-facing send step is considered.

---

## Portfolio Screenshots

### Workflow Overview

<img width="1685" height="326" alt="image" src="https://github.com/user-attachments/assets/a0000b17-97f5-4b15-ab89-a86ec2bde83c" />

### Airtable Review Output

<img width="1260" height="632" alt="image" src="https://github.com/user-attachments/assets/81200018-7df4-47dd-84d6-7e6d4bcec0f8" />

### Zapier Trigger Setup

<img width="416" height="385" alt="image" src="https://github.com/user-attachments/assets/1a27b6d0-bbf0-4aa9-a22e-0458294c1091" />

---

## Test Notes

The workflow has been tested end-to-end on pickleball-focused partner categories and general control companies.

Observed examples include:

| Example Company Type | Category | Partner Group | Expected Result |
|---|---|---|---|
| Pickleball governing body | pickleball governing body | program_partner | high-confidence fit |
| Pickleball equipment brand | pickleball equipment | brand_partner | high-confidence fit |
| Pickleball media company | pickleball media | program_partner | high-confidence fit |
| Pickleball travel or experience company | travel and experiences | experience_partner | high-confidence fit |
| Sports facility software vendor | facility software | vendor_partner | moderate to strong fit |
| Activewear/lifestyle apparel brand | sports apparel | brand_partner | moderate fit |
| General productivity software control | software | brand_partner | weak fit |

Recent validation runs confirmed:

- execution `1059` completed successfully in 50 seconds with zero node errors
- a publishable assessment can produce ranking, outreach, contact-role, Airtable, MongoDB, and owner-email outputs
- a 5-company sequential execution can complete through the queue loop
- publishable assessments generate outreach drafts and contact-role records
- outreach drafts save to MongoDB and Airtable review tables
- owner-review emails can send automatically through SMTP/Gmail when credentials are configured
- sender-domain inputs from Zapier can flow into outreach readiness fields
- manual outreach status and follow-up fields are available for review-first sending
- failed-provider assessments are blocked from outreach until a publishable assessment exists

These examples are prototype test results derived from public-web data. They do not represent official recommendations, endorsements, or affiliations with any listed organization.

---

## What This Demonstrates

- Designing stateful automation instead of one-off scraping
- Building retry-aware workflows around unreliable external services
- Combining AI classification with deterministic validation and scoring
- Turning messy public-web evidence into review-ready business outputs
- Preparing outbound-ready prospect intelligence before automated sending
- Generating company-context email drafts from assessment evidence
- Sending automatic owner-review notifications while gating prospect-facing outreach
- Separating operational state in MongoDB from human review workflows in Airtable

---

## Key Capabilities

- Zapier-compatible webhook intake
- Optional Apollo company discovery
- Optional Clay enrichment handoff
- MongoDB-backed queueing and retry state
- Sequential multi-company assessment runs
- Reduced workflow complexity from 119 to 114 nodes after adding the owner-review email node
- Company/root-domain dedupe
- Public-web page discovery and fetch validation
- Blocked, empty, soft-404, and suspicious-domain fallback handling
- Multi-page company context extraction
- AI-assisted page classification
- Deterministic scoring and validation guardrails
- Provider-failure recovery and retryable job states
- Page-level evidence storage
- Company-level sponsor-fit aggregation
- Short-term and long-term partnership potential scoring
- Airtable review outputs for assessments, rankings, contacts, and outreach
- Company-context outreach draft generation
- Automatic owner-review email notification for generated outreach drafts
- Manual outreach tracking and sender-domain readiness fields
- Early rerun comparison and change-detection support

---

## Roadmap

- Stabilize model routing and compare Gemma/Gemini variants for reliability, speed, and quota efficiency
- Consider simplifying five classification batch branches into one looped classifier lane
- Add automated retry scheduling for `failed_provider` and `needs_retry` jobs
- Add stronger run-level observability for current node, provider failures, and token/request burn
- Further simplify MongoDB queue seed records into a minimal input schema
- Improve automated discovery of company websites when only a company name is provided
- Reduce AI calls with tighter pre-AI page selection and text limits
- Add material-change detection for repeated brand scans
- Add stronger upsert/dedupe behavior for MongoDB and Airtable records
- Move more category, guardrail, and controlled-value logic into MongoDB taxonomy/runtime config
- Add person/contact enrichment only after a prospect reaches a review threshold
- Add automated outbound domain-readiness checks for SPF, DKIM, and DMARC before campaign launch
- Add Smartlead or Instantly handoff after human approval and verified deliverability readiness

---

## Design Principles

- Multi-source evidence over single-page assumptions
- Page-level evidence before company-level conclusions
- Deterministic validation over blind AI trust
- Enrichment metadata as context, not final truth
- Provider failures should be retryable, not silently published
- Review-first outreach before prospect-facing send automation
- MongoDB for memory and operational state
- Airtable for review and action
- Conservative fallback behavior when AI output is missing or weak

---

## Potential Use Cases

- Sponsor prospect qualification
- Brand fit scoring
- Partnership target discovery
- Outreach prioritization
- Sponsorship category research
- Manual outreach preparation
- Historical monitoring of public commercial signals
- Sport-level partnership intelligence
- Ecosystem-level sponsor discovery

---

## Why This Matters

This is more than a scraping workflow.

The project combines webhook-triggered automation, prospect discovery, public-web extraction, AI-assisted classification, deterministic scoring, retryable operational state, structured review outputs, and review-first outreach preparation.

The goal is to help identify which organizations and brands are actually worth reviewing, ranking, and prioritizing for partnership outreach.
