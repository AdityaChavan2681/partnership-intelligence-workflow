# Partnership Intelligence Pipeline

Brand prospect assessment, sponsor-fit scoring, and public-web partnership intelligence for sports partnership teams.

> Status: Active development. This repository currently documents the project architecture, workflow behavior, and implementation notes. The n8n workflow export, credentials, and environment-specific configuration are not included yet.

This project uses n8n to evaluate companies as potential sponsors, partners, vendors, media partners, or activation partners for a target sports market. The current workflow is tuned for the pickleball ecosystem, but the intake fields and scoring model are designed to support other sports, events, venues, leagues, and commercial partnership categories.

The system is designed to answer:

```text
Given a target sport, league, event, venue, or sports property and an outside company,
is this company a credible sponsor or partnership prospect?
```

---

## Repository Scope

This repository is currently README-only while the workflow is being developed and tested. It is intended to document the system design, current workflow behavior, data flow, integrations, and operating lessons before the n8n export and setup assets are prepared for release.

The working n8n workflow, local credentials, API keys, Zapier setup, MongoDB connection details, Airtable configuration, and temporary Cloudflare Tunnel URLs are intentionally excluded.

Local testing and debugging may use PowerShell commands for n8n API calls, execution-output exports, and webhook/tunnel checks.

---

## Overview

Partnership teams often evaluate prospects from scattered public signals: homepages, product pages, partner pages, athlete pages, event pages, blog posts, news pages, media pages, and contact pages. Those signals are valuable, but they are noisy and hard to compare manually.

This pipeline converts public-web evidence into structured sponsor-prospect intelligence that can be queued, scored, reviewed, retried, stored, and monitored over time.

The workflow combines:

- Zapier-compatible webhook intake
- n8n workflow orchestration
- MongoDB-backed queueing, dedupe, retry state, evidence storage, and run status
- Optional Apollo discovery when credits are available
- Optional Clay enrichment handoff
- Public website discovery and text extraction
- Gemma/Gemini-compatible model nodes for company context and page classification
- Deterministic scoring, normalization, and quality gates
- Airtable review outputs for assessments, rankings, contacts, and outreach drafts

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
→ update MongoDB job status and pipeline run status
```

---

## Workflow State

Current exported workflow file:

```text
partnership-intelligence-workflow-revamped.json
```

Workflow snapshot:

- 113 n8n nodes after revamp, reduced from 119 nodes in the previous export
- Removed 7 wait nodes while preserving sequential company processing and extractor retry control
- Zapier-compatible webhook path: `partnership-pipeline-run`
- MongoDB queue collection: `brand_assessment_jobs`
- Page evidence collection: `brand_page_assessments`
- Company assessment collection: `brand_assessments`
- Outreach draft collection: `outreach_drafts`
- Pipeline status collections: `pipeline_run_status`, `pipeline_run_events`
- Current model routing: `models/gemma-4-26b-a4b-it`
- Extractor retries: 2 tries per extractor node

The main assessment still processes companies sequentially. `max_companies_per_run` controls how many queued companies can be selected for a run, with the workflow capped at 5.

---

## Data Flow

The pipeline starts from a webhook-triggered run request. Zapier is the current intended trigger, but any tool that can send an HTTP POST can start the workflow.

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
```

Apollo is useful for discovery, but it is not required for assessment execution. If Apollo credits are unavailable, the workflow can still assess companies already present in MongoDB.

MongoDB is the operational store. Airtable is the review layer.

---

## Webhook Run Input

Zapier currently sends fields like this:

```json
{
  "search_keywords": "pickleball equipment",
  "search_location": "United States",
  "search_page_size": 5,
  "search_page_count": 10,
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
  "max_companies_per_run": 5
}
```

Notes:

- `search_page_count` controls Apollo discovery pagination. It does not control website crawl depth.
- Website crawl limits are controlled by runtime config, including `crawl_config.max_pages_total`.
- `search_page_size` controls the Apollo page size when Apollo discovery is enabled.
- `max_companies_per_run` controls how many queued companies can be processed sequentially in one run.
- During testing, `max_companies_per_run: 1` is recommended to avoid burning model quota while debugging.

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

Scores are based on public-web evidence and support both short-term activation opportunities and long-term partnership evaluation. The workflow rewards signals such as sport/category relevance, sponsor fit, product or service alignment, partnership language, event/community activation potential, evidence breadth, and source quality.

The scoring logic combines page-level evidence scores with brand-level aggregation and quality gates. AI is used to classify and summarize evidence, while deterministic workflow logic handles scoring, caps, retry status, timeline potential, and validation.

The workflow also produces separate short-term and long-term partnership potential fields. Short-term potential reflects immediate campaign, event, media, product, venue, or activation signals. Long-term potential reflects durable sponsor fit, partnership language, evidence breadth, high-fit pages, and source quality.

---

## Main Outputs

The workflow produces five review layers:

- Assessment: score, conclusion, recommended action, category, business model, and offer type
- Timeline: short-term partnership potential, long-term partnership potential, timeline recommendation, and rationale
- Evidence: source URLs, top evidence pages, evidence summaries, page counts, and signal flags
- Reliability: quality status, retry flag, provider-error count, fallback count, blocked-page count, and evidence strength
- Ranking: rank tier, reliability level, strategic fit, commercial fit, activation fit, and ranking rationale
- Action: contact-role targets and outreach draft fields

---

## Reliability Model

The workflow is designed to avoid publishing weak or failed assessments as successful partner recommendations.

Current reliability behavior:

- Apollo duplicate-only runs can continue into existing queued or retryable MongoDB jobs.
- Apollo provider errors or credit limits do not block existing MongoDB queue assessment.
- Website evidence can override stale or incomplete enrichment metadata.
- Blocked, empty, failed, and soft-404 pages produce fallback records instead of disappearing silently.
- Suspicious or hijacked-looking domains can be rejected deterministically before wasting model calls.
- Model/provider failures are marked as `failed_provider` or `needs_retry` instead of being published as completed assessments.
- Quality gates prevent provider-error assessments from being treated as review-ready results.
- Reruns can suppress unchanged Airtable outputs when score, evidence shape, and recommendation do not materially change.

Current model note:

The workflow is currently routed through `models/gemma-4-26b-a4b-it` because earlier Gemini Flash Lite tests frequently returned service availability errors. The extractor nodes are capped at 2 retries to reduce long retry loops and quota burn. Provider-side `GoogleGenerativeAIError` / service-unavailable errors are treated as model availability issues, not prompt or scoring failures.

---

## Runtime and Model Notes

Model choice matters because each company can create several company-context and page-classification calls.

Operating notes:

- The current workflow uses `models/gemma-4-26b-a4b-it` because it has been more stable than `gemini-3.1-flash-lite` during recent extractor tests.
- Gemma 4 26B has lower RPM than some Gemma 3 variants, but its unlimited TPM is useful for larger page-text inputs.
- Gemma 3 variants have higher RPM/RPD and may be worth testing later, but their 15K TPM limit means page text may need tighter trimming.
- Gemini Flash Lite has high TPM, but repeated service-unavailable errors made it less reliable for this workflow during testing.
- RPM, TPM, and RPD are the practical throughput limits; sampling settings are kept conservative for consistent structured extraction.

---

## MongoDB Collections

MongoDB stores operational state, evidence, and historical assessment records.

Collections include:

- `brand_assessment_jobs`: lightweight queue, dedupe state, job status, retry status
- `brand_page_assessments`: page-level evidence and page scores
- `brand_assessments`: company-level assessment history
- `outreach_drafts`: generated outreach draft records
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

The `brand_assessments` table stores company-level sponsor-fit assessments. `partnership_prospect_rankings` stores prioritization records with score, fit, evidence strength, reliability level, and ranking rationale. `prospect_contacts` stores recommended contact-role placeholders and future person-level enrichment records. `outreach_review_queue` stores review-ready outreach drafts.

---

## Integrations

### Zapier

Zapier is the current trigger layer. It sends the run request to the n8n webhook URL and supplies search, targeting, scoring-priority, and run-size fields.

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

---

## Review Output Contract

The main output is a review-ready company-level partnership assessment written to MongoDB and Airtable.

```json
{
  "prospect_name": "USA Pickleball",
  "target_organization": "Pickleball",
  "root_domain": "https://usapickleball.org",
  "partner_group": "program_partner",
  "company_category": "pickleball governing body",
  "business_model": "b2b services",
  "recommended_offer": "corporate sponsorship",
  "brand_fit_score": 100,
  "rank_tier": "A - prioritize",
  "recommended_action": "prioritize outreach",
  "short_term_partnership_potential": "high",
  "short_term_partnership_potential_score": 88,
  "long_term_partnership_potential": "high",
  "long_term_partnership_potential_score": 96,
  "partnership_timeline_recommendation": "long-term partnership",
  "reliability_level": "high confidence",
  "evidence_strength": "strong",
  "evidence_page_count": 9,
  "strong_evidence_page_count": 9,
  "primary_partnership_angle": "National-level brand integration, tournament visibility, sanctioning, and official partnership opportunities.",
  "primary_evidence_summary": "Public pages identify USA Pickleball as a national governing body with tournament, membership, and partnership signals.",
  "ranking_reason": "Rank A from score 100 with strong evidence across multiple pages and clear sport-specific partnership relevance.",
  "expected_value_driver": "event sponsorship and tournament visibility",
  "strategic_fit": "high",
  "commercial_fit": "high",
  "activation_fit": "medium",
  "relationship_depth": "campaign or event partner",
  "assessment_quality_status": "completed",
  "assessment_date": "2026-05-10"
}
```

A separate `partnership_prospect_rankings` output translates the assessment into a prioritization record for sorting prospects by fit, reliability, evidence strength, partnership timeline potential, and recommended next action.

---

## Portfolio Screenshots

### Workflow Overview

<img width="1491" height="259" alt="image" src="https://github.com/user-attachments/assets/95908db4-0aa6-4493-8dbb-1eff0c93465c" />

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

Recent model testing showed that `gemini-3.1-flash-lite` could return frequent service-unavailable errors under load. `gemma-4-26b-a4b-it` has been more stable in the workflow, though still slower and subject to provider availability.

A recent 5-company execution completed successfully after routing the workflow through the more stable Gemma model path and reducing extractor retry loops. This validated the sequential queue behavior across multiple companies, but runtime and provider reliability still depend heavily on model availability, page count, extracted text volume, and retry settings.

These examples are prototype test results derived from public-web data. They do not represent official recommendations, endorsements, or affiliations with any listed organization.

---

## What This Demonstrates

- Designing stateful automation instead of one-off scraping
- Building retry-aware workflows around unreliable external services
- Combining AI classification with deterministic validation and scoring
- Turning messy public-web evidence into review-ready business outputs
- Separating operational state in MongoDB from human review workflows in Airtable

---

## Key Capabilities

- Zapier-compatible webhook intake
- Optional Apollo company discovery
- Optional Clay enrichment handoff
- MongoDB-backed queueing and retry state
- Sequential multi-company assessment runs
- Reduced workflow complexity from 119 to 113 nodes
- Company/root-domain dedupe
- Public-web page discovery and fetch validation
- Blocked, empty, soft-404, and suspicious-domain fallback handling
- Multi-page company context extraction
- AI-assisted page classification
- Deterministic scoring and validation guardrails
- Provider-failure recovery and retryable job states
- Page-level evidence storage
- Company-level sponsor-fit aggregation
- Airtable review outputs for assessments, rankings, contacts, and outreach
- Early rerun comparison and change-detection support

---

## Roadmap

- Stabilize model routing and compare Gemma 3/Gemma 4/Gemini variants for reliability, speed, and quota efficiency
- Add automated retry scheduling for `failed_provider` and `needs_retry` jobs
- Add stronger run-level observability for current node, provider failures, and token/request burn
- Further simplify MongoDB queue seed records into a minimal input schema
- Improve automated discovery of company websites when only a company name is provided
- Reduce AI calls with tighter pre-AI page selection and text limits
- Add material-change detection for repeated brand scans
- Add stronger upsert/dedupe behavior for MongoDB and Airtable records
- Move more category, guardrail, and controlled-value logic into MongoDB taxonomy/runtime config
- Add person/contact enrichment only after a prospect reaches a review threshold
- Add Smartlead or Instantly send-step integration after human approval

---

## Design Principles

- Multi-source evidence over single-page assumptions
- Page-level evidence before company-level conclusions
- Deterministic validation over blind AI trust
- Enrichment metadata as context, not final truth
- Provider failures should be retryable, not silently published
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
- Historical monitoring of public commercial signals
- Sport-level partnership intelligence
- Ecosystem-level sponsor discovery

---

## Why This Matters

This is more than a scraping workflow.

The project combines webhook-triggered automation, optional prospect discovery, public-web extraction, AI-assisted classification, deterministic scoring, retryable operational state, and structured review outputs.

The goal is to help identify which organizations and brands are actually worth reviewing, ranking, and prioritizing for partnership outreach.
