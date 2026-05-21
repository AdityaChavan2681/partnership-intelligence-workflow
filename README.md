# Partnership Intelligence Pipeline

Brand prospect assessment, sponsor-fit scoring, decision support, and review-first outreach preparation for sports partnership teams.

> Status: Active development. This repository currently documents the project architecture, workflow behavior, screenshots, and implementation notes. Credentials, API keys, database connection strings, local tunnel URLs, and environment-specific configuration are intentionally excluded.

> Latest verified run: A Zapier-triggered/local n8n pipeline execution completed successfully. A decision-gating test assessed `Canva`, produced a company-level decision of `monitor`, saved the decision to Airtable, skipped outreach draft generation, and marked the pipeline run complete.

This project uses n8n to evaluate companies as potential sponsors, partners, vendors, media partners, facility partners, program partners, or activation partners for a target sports market. The current workflow is tuned for the pickleball ecosystem, but the intake fields and scoring model are designed to support other sports, events, venues, leagues, and commercial partnership categories.

The system is designed to answer:

```text
Given a target sports market and an outside company,
is this company worth pursuing, reviewing, monitoring, or rejecting as a partnership prospect?
```

---

## Repository Scope

This repository is README-first while the workflow is being developed and tested. It documents the system design, workflow behavior, integrations, operating lessons, and portfolio screenshots before the n8n export and setup assets are prepared for release.

The working n8n workflow, local credentials, API keys, Zapier setup, MongoDB connection details, Airtable credentials, SMTP credentials, and temporary Cloudflare Tunnel URLs are intentionally excluded.

Local testing and debugging may use PowerShell commands, n8n API calls, execution-output exports, and webhook/tunnel checks.

---

## Overview

Partnership teams often evaluate prospects from scattered public signals: homepages, product pages, partner pages, athlete pages, event pages, blog posts, media pages, and contact pages. Those signals are useful, but they are noisy and hard to compare manually.

This pipeline converts public-web evidence into structured sponsor-prospect intelligence that can be queued, scored, reviewed, retried, stored, and turned into outreach-ready drafts.

The workflow combines:

- Zapier-compatible webhook intake
- n8n workflow orchestration
- MongoDB-backed queueing, dedupe, retry state, evidence storage, and run status
- Optional Apollo discovery when credits are available
- Optional Clay enrichment handoff
- Public website discovery and text extraction
- Gemini-backed company context and page classification
- A single looped AI page-classification lane
- Deterministic scoring, normalization, and quality gates
- Company-level decision support: `pursue`, `review`, `monitor`, or `reject`
- Airtable decision and outreach review outputs
- Review-first outreach preparation with automatic owner-review email notifications
- Manual outreach tracking and sender-domain readiness fields

MongoDB is the operational system of record. Airtable is the human review layer.

---

## Architecture

```text
Zapier or HTTP webhook trigger
-> n8n pipeline run webhook
-> pipeline run lock/status check
-> optional Apollo prospect discovery
   -> if Apollo succeeds: normalize and queue discovered companies in MongoDB
   -> if Apollo has no credits or fails: continue from the MongoDB queue
   -> if Clay handoff is configured: optionally send enrichment payloads
-> select one queued, retryable, provider-failed, or stale-running assessment job from MongoDB
-> claim/update selected assessment job state
-> fetch homepage with browser-like headers
-> discover internal links and fallback probe URLs
-> filter and classify relevant website pages
-> fetch selected pages and validate HTTP/page quality
-> extract clean page text and preserve evidence snippets
-> build multi-page company context input
-> extract company-level context
-> attach company context to page-level records
-> loop through selected pages one at a time
   -> preserve source metadata
   -> classify page-level partnership signal with Gemini extractor
   -> normalize model output and apply page guardrails
-> merge classified pages with blocked-page fallback records
-> validate page classifications
-> score page-level sponsor-fit evidence
-> store page-level evidence in MongoDB
-> aggregate brand-level assessment
-> prepare prospect review decision
-> save company assessment to MongoDB
-> save decision record to Airtable
-> generate company-context outreach draft
-> save outreach draft to MongoDB and Airtable
-> create contact placeholder/review records
-> send owner-review email through SMTP/Gmail when enabled
-> update MongoDB job status
-> mark pipeline run complete
```

---

## Workflow State

Current workflow snapshot:

- 106 n8n nodes including disabled legacy/reference nodes
- Active Zapier-compatible webhook path: `partnership-pipeline-run`
- MongoDB queue collection: `brand_assessment_jobs`
- Page evidence collection: `brand_page_assessments`
- Company assessment collection: `brand_assessments`
- Outreach draft collection: `outreach_drafts`
- Pipeline status collections: `pipeline_run_status`, `pipeline_run_events`
- Current model routing: `models/gemini-3.1-flash-lite`
- Owner-review email sender used in testing: `n8nemailtestpartner@gmail.com`
- Default stale lock timeout for development/manual runs: 10 minutes
- Manual/dev pipeline run size: 1 company per run
- Page classification uses one looped extractor lane instead of duplicated parallel batch branches

The workflow currently processes one selected company per manual or Zapier-triggered development run. This keeps test runs predictable, reduces nested-loop completion issues, and still allows repeated queue processing through separate runs.

---

## Webhook Run Input

Zapier can send fields like this:

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
  "pipeline_stale_after_minutes": 10,
  "outreach_delivery_platform": "review_only",
  "sender_domain": "testdomain.com",
  "dkim_selector": "default",
  "manual_sender_email": "test-sender@example.com",
  "owner_review_email_to": "reviewer@example.com"
}
```

Notes:

- `search_page_count` controls Apollo discovery pagination. It does not control website crawl depth.
- `search_page_size` controls Apollo page size when Apollo discovery is enabled.
- Website crawl limits are controlled by runtime config, including `crawl_config.max_pages_total`.
- `max_companies_per_run` is currently kept at `1` for stable manual and Zapier-driven development runs.
- `pipeline_stale_after_minutes` can override the default lock timeout, capped between 1 and 90 minutes.
- `manual_sender_email` and `owner_review_email_to` are runtime-configurable and should use placeholders in public documentation.
- Prospect-facing sends are not automatic. The current delivery mode is review-first.

---

## Decision Layer

After brand-level aggregation, the workflow prepares a prospect review decision.

Decision outputs include:

- `review_decision`: `pursue`, `review`, `monitor`, or `reject`
- `decision_confidence`: `high`, `medium`, or `low`
- `best_use_case`
- `why_this_company`
- `main_risk`
- `next_human_step`
- `priority_reason`
- `disqualifier_reason`
- `decision_version`

This layer turns raw assessment scores into a human-readable decision record. The goal is not only to say that a company scored well, but to explain what to do next, why it stands out, and what risk should be checked before outreach.

Decision examples:

```text
pursue  -> strong fit, strong evidence, clear use case
review  -> promising fit, but needs human judgment or more verification
monitor -> some relevance, but not enough evidence for outreach yet
reject  -> weak fit, unsafe domain, failed quality checks, or very low score
```

---

## Brand Assessment Workflow

The Brand Prospect Assessor evaluates an outside company as a potential sponsor or partner for a target sports market.

Starting from a company name and root domain, the workflow discovers and evaluates relevant pages such as:

- homepage
- about pages
- product and category pages
- partner, sponsorship, affiliate, ambassador, or collaboration pages
- blog, news, media, and press pages
- event, athlete, club, facility, or tournament pages
- contact pages as supporting context only

Each page is classified and scored individually. The workflow then aggregates the strongest page-level evidence into one company-level sponsor-fit assessment.

Before page-level classification, the workflow builds a multi-page company context record. This helps identify what the analyzed organization primarily does while keeping final scoring grounded in each page's own evidence.

---

## AI Classification Refactor

The page-classification section was simplified from five duplicated batch branches into a single looped classifier lane.

Previous shape:

```text
Route Pages By AI Batch
-> Preserve Source Metadata Batch 1-5
-> Classify Partnership Signal Batch 1-5
-> Normalize AI Batch 1-5
-> Merge AI Classified Pages
```

Current shape:

```text
Prepare AI Classification Items
-> Loop AI Page Classification
   -> Preserve Source Metadata
   -> Classify Partnership Signal
   -> Normalize AI Output
-> Merge AI Classified Pages
-> Apply Page Classification Guardrails
```

Benefits:

- fewer n8n nodes
- one prompt to maintain
- one normalization code path
- easier debugging in execution output
- lower risk of branch drift between classifier copies
- same page-level output shape for downstream scoring and aggregation

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

The outreach layer prepares drafts and review records before any prospect-facing send step.

A completed publishable assessment can generate:

- a company-context outreach draft
- a suggested subject line and email body
- evidence used for personalization
- recommended contact roles
- manual outreach status fields
- sender-domain readiness fields for SPF, DKIM, and DMARC
- an automatic owner-review email through SMTP/Gmail when credentials are configured

The owner-review email is controlled by webhook/Zapier runtime fields:

```text
manual_sender_email
owner_review_email_to
```

Prospect-facing sends are intentionally gated behind human review, verified contact data, sender-domain readiness, and future platform approval rules.

Smartlead, Instantly, or similar outbound platforms remain planned as optional downstream delivery layers after human approval. The current workflow is review-first rather than bulk-send-first.

---

## Main Outputs

The workflow produces four active review layers:

- Decision: pursue/review/monitor/reject, confidence, best use case, risk, rationale, and next human step
- Assessment: score, conclusion, recommended action, category, business model, offer type, and evidence summaries
- Reliability: quality status, retry flag, provider-error count, fallback count, blocked-page count, and evidence strength
- Outreach: draft email, owner-review notification, manual outreach status, sender-domain readiness, and contact-role targets

Operational/debug details stay in MongoDB. Airtable is kept focused on human review and action.

---

## Reliability Model

The workflow is designed to avoid publishing weak or failed assessments as successful partner recommendations.

Reliability behavior:

- Pipeline run locks prevent overlapping runs.
- Stale locks are ignored after the configured timeout.
- Completed runs store processed, completed, retryable, and failed-provider job counts.
- Apollo duplicate-only runs can continue into existing queued or retryable MongoDB jobs.
- Apollo provider errors, credit limits, or network errors do not block existing MongoDB queue assessment.
- Website evidence can override stale or incomplete enrichment metadata.
- Blocked, empty, failed, and soft-404 pages produce fallback records instead of disappearing silently.
- Suspicious or hijacked-looking domains can be rejected deterministically before wasting model calls.
- Model/provider failures are marked as `failed_provider` or `needs_retry` instead of being published as completed assessments.
- Quality gates prevent provider-error assessments from being treated as review-ready results.
- Failed-provider assessments are blocked from outreach until a publishable assessment exists.

---

## Runtime and Model Notes

Model choice matters because each company can create one company-context extraction and multiple page-classification calls.

Current Gemini extractor settings:

```text
Model: models/gemini-3.1-flash-lite
Max output tokens: 600
Temperature: 0.1
Top K: 20
Top P: 0.8
```

These conservative settings are intended to keep extractor outputs short, structured, and repeatable. The workflow still applies deterministic guardrails after AI output, so model settings are not treated as the only reliability layer.

Current operating notes:

- Company-context and page-classification calls use `models/gemini-3.1-flash-lite`.
- Page classification runs sequentially through a loop rather than five parallel branches.
- Sequential page classification is slower than parallel extraction, but it is easier to debug, cheaper to maintain, and more predictable for manual runs.

---

## AI Quota Telemetry

The workflow records estimated Gemini request usage for each completed pipeline run.

Current telemetry is based on workflow node counts rather than provider-reported token usage. n8n does not currently expose exact Gemini token consumption or live remaining Google AI Studio quota in this workflow.

The current Google AI Studio quota envelope used for planning is:

```text
Gemini 3.1 Flash Lite
RPM: 15
TPM: 250K
RPD: 500
```

For each completed run, the pipeline stores fields such as:

- `ai_provider`
- `ai_model`
- `ai_company_context_call_count`
- `ai_page_classification_call_count`
- `ai_total_estimated_call_count`
- `ai_classified_page_count`
- `ai_provider_error_count`
- `quota_rpd_limit`
- `quota_estimated_rpd_used_this_run`
- `quota_estimated_company_capacity_at_limit`
- `quota_safety_status`
- `quota_notes`

These fields are stored on the latest pipeline run status and cleared when a new run starts. This keeps resource telemetry operational rather than mixing it into Airtable review outputs.

---

## MongoDB Collections

MongoDB stores operational state, evidence, outreach drafts, and historical assessment records.

Collections include:

- `brand_assessment_jobs`: lightweight queue, dedupe state, job status, retry status
- `brand_page_assessments`: page-level evidence and page scores
- `brand_assessments`: company-level assessment history and decision fields
- `outreach_drafts`: outreach drafts, owner-review email payloads, manual outreach status, sender-domain readiness
- `config`: runtime config, taxonomy, and crawl settings
- `pipeline_run_status`: run lock and current run status
- `pipeline_run_events`: skip/no-work/status events

The queue is intentionally lightweight. Detailed evidence, scoring, decision, and outreach data are stored downstream after assessment.

---

## Airtable Review Architecture

Airtable is used for human review, filtering, prioritization, and approval.

Active review tables:

- `prospect_decisions`: main company-level decision board
- `outreach_review_queue`: generated outreach drafts pending review
- `prospect_contacts`: contact and decision-maker tracking

Inactive/future table:

- `company_enrichment`: kept idle as a future Clay/Apollo/manual enrichment cache

Deprecated Airtable branches were disabled and moved into a separate `Legacy Disabled Airtable Branches` area on the n8n canvas. They are retained as rollback/reference only and are no longer targeted by active connections.

Disabled legacy Airtable branches include:

- old brand assessment Airtable output
- old assessment history Airtable output
- previous-assessment comparison review update
- old prospect ranking Airtable output

This keeps Airtable focused on the product workflow:

```text
Should we pursue this company?
What is the risk?
What is the best use case?
What should a human do next?
What outreach draft needs review?
Who might we contact?
```

---

## Integrations

### Zapier

Zapier is the current trigger layer. It sends the run request to the n8n webhook URL and supplies search, targeting, scoring-priority, run-size, and outreach-readiness fields.

### Cloudflare Tunnel

Cloudflare Tunnel is used during local development to expose the local n8n webhook to external tools such as Zapier without deploying the workflow to a public server. Tunnel URLs are temporary development endpoints and are not included in the repository.

### Apollo

Apollo is optional discovery. It can supply prospect companies, domains, industry metadata, location data, employee estimates, LinkedIn URLs, and keyword signals. If Apollo credits are exhausted or Apollo has a provider/network error, the assessment pipeline can continue from MongoDB queue records.

### Clay

Clay is an optional enrichment/reference-data handoff. n8n can send company payloads to Clay when enrichment is needed, while the main assessment continues without requiring a Clay callback.

### MongoDB

MongoDB stores queue state, dedupe state, retry status, page-level evidence, company-level assessment history, pipeline run status, and outreach draft records.

### Airtable

Airtable stores decision and outreach review records for sorting, filtering, prioritization, and approval.

### SMTP/Gmail

SMTP/Gmail can send automatic owner-review notifications after an outreach draft is generated during a workflow run. Testing used a dedicated test inbox rather than a personal address. Public documentation should use placeholder emails.

### Outreach Delivery Tools

Smartlead, Instantly, or similar outbound platforms are planned as optional downstream delivery layers after human review. Prospect-facing sends are intentionally gated behind approval, verified contact data, and sender-domain readiness.

---

## Verified Output Example

The latest exported decision-gating test completed successfully after the outreach gate was tightened to generate drafts only for `pursue` decisions.

Verified execution summary:

```text
Status: success
Started at: 2026-05-21T14:11:13.001Z
Stopped at: 2026-05-21T14:11:49.053Z
Measured runtime: 36.1 seconds
Top-level errors: 0
Company: Canva
Root domain: https://www.canva.com
Pipeline completion status: completed
```

Company decision output:

```json
{
  "analyzed_organization": "Canva",
  "root_domain": "https://www.canva.com",
  "brand_fit_score": 47,
  "brand_fit_conclusion": "moderate brand fit",
  "recommended_action": "monitor or enrich",
  "review_decision": "monitor",
  "decision_confidence": "medium",
  "best_use_case": "vendor partnership or operational sponsor",
  "main_risk": "Evidence is thin; review sources before approving outreach.",
  "next_human_step": "Keep the prospect in the database and enrich it with stronger evidence before outreach.",
  "assessment_quality_status": "complete",
  "needs_retry": false
}
```

Outreach gate output:

```json
{
  "prepare_outreach_draft_items": 0,
  "save_outreach_draft_to_mongodb": "not_run",
  "save_outreach_draft_to_airtable": "not_run",
  "send_owner_review_email": "not_run"
}
```

Pipeline completion output:

```json
{
  "status": "completed",
  "processed_job_count": 1,
  "completed_job_count": 1,
  "retryable_job_count": 0,
  "failed_provider_job_count": 0
}
```

The outreach branch creates review-ready email drafts, owner-review email notifications, manual outreach status fields, and contact-role placeholders before any prospect-facing send step is considered.

Only `pursue` decisions generate outreach drafts automatically. `monitor` and `reject` decisions are saved to the decision board without creating an outreach draft or owner-review email.

---

## Decision Spectrum Validation

Recent test runs were used to confirm that the workflow can distinguish strong, adjacent, middle, and weak prospects instead of blindly rewarding brand recognition.

| Company | Fit Pattern | Score | Decision | Outreach Draft |
|---|---:|---:|---|---|
| Recess Pickleball | direct pickleball product fit | 100 | pursue | yes |
| Liquid I.V. | adjacent hydration/event activation fit | 100 | pursue | yes |
| Canva | middle vendor/content support fit | 47 | monitor | no |
| Microsoft | weak/non-sport-specific fit | 6 | reject | no |

The Microsoft test is important because it shows that a large, well-known company can still be rejected when the public evidence does not support a credible pickleball partnership angle. The Canva test is important because it shows a moderate operational/content use case can be retained for monitoring without triggering outreach.

---

## Portfolio Screenshots

### Workflow Overview

<img width="1524" height="256" alt="image" src="https://github.com/user-attachments/assets/ea89effc-f9d1-41ac-8c29-66d9944bb768" />

### Airtable Review Output

<img width="1260" height="632" alt="image" src="https://github.com/user-attachments/assets/81200018-7df4-47dd-84d6-7e6d4bcec0f8" />

### Zapier Trigger Setup

<img width="416" height="385" alt="image" src="https://github.com/user-attachments/assets/1a27b6d0-bbf0-4aa9-a22e-0458294c1091" />

---

## Test Notes

Recent validation runs confirmed:

- a full Zapier-triggered/local pipeline run completed successfully with zero top-level errors
- the latest decision-gating run completed in 36.1 seconds
- Apollo insufficient-credit responses are detected and skipped without blocking assessment
- Apollo network/DNS/provider errors are configured to continue into the skip path
- the AI page classifier loop processes selected pages through a single reusable lane
- publishable assessments generate MongoDB assessment records, Airtable decision records, outreach drafts, contact-role records, and owner-email outputs
- `monitor` decisions save to `prospect_decisions` without generating outreach drafts
- `reject` decisions save to `prospect_decisions` without generating outreach drafts
- only `pursue` decisions currently generate outreach drafts and owner-review emails
- owner-review email configuration now comes from Zapier/webhook runtime fields
- the pipeline run lock starts as `running` and finishes as `completed`
- outreach drafts save to MongoDB and Airtable review tables
- prospect decision records save to the `prospect_decisions` Airtable table
- sender-domain inputs from Zapier flow into outreach readiness fields
- manual outreach status and follow-up fields are available for review-first sending
- failed-provider assessments are blocked from outreach until a publishable assessment exists

These examples are prototype test results derived from public-web data. They do not represent official recommendations, endorsements, or affiliations with any listed organization.

---

## What This Demonstrates

- Designing stateful automation instead of one-off scraping
- Building retry-aware workflows around unreliable external services
- Combining AI classification with deterministic validation and scoring
- Turning messy public-web evidence into review-ready business outputs
- Adding decision support on top of raw scoring
- Preparing outbound-ready prospect intelligence before automated sending
- Generating company-context email drafts from assessment evidence
- Sending automatic owner-review notifications while gating prospect-facing outreach
- Separating operational state in MongoDB from human review workflows in Airtable
- Refactoring duplicated AI branches into a maintainable loop without changing downstream outputs
- Keeping public documentation safe by using placeholder runtime email values

---

## Key Capabilities

- Zapier-compatible webhook intake
- Optional Apollo company discovery
- Optional Clay enrichment handoff
- MongoDB-backed queueing and retry state
- One-company manual pipeline runs for stable development execution
- Pipeline run lock with stale timeout and completion status
- Company/root-domain dedupe
- Public-web page discovery and fetch validation
- Blocked, empty, soft-404, and suspicious-domain fallback handling
- Multi-page company context extraction
- Looped AI-assisted page classification
- Deterministic scoring and validation guardrails
- Provider-failure recovery and retryable job states
- Page-level evidence storage
- Company-level sponsor-fit aggregation
- Short-term and long-term partnership potential scoring
- Prospect review decision layer
- Airtable decision board output
- Company-context outreach draft generation
- Automatic owner-review email notification for generated outreach drafts
- Manual outreach tracking and sender-domain readiness fields

---

## Roadmap

- Keep validating the single-loop page classifier across broader company categories
- Add automated retry scheduling for `failed_provider` and `needs_retry` jobs
- Add stronger run-level observability for current node, provider failures, and request usage
- Further simplify MongoDB queue seed records into a minimal input schema
- Improve automated discovery of company websites when only a company name is provided
- Reduce AI calls with tighter pre-AI page selection and text limits
- Add person/contact enrichment only after a prospect reaches a review threshold
- Add automated outbound domain-readiness checks for SPF, DKIM, and DMARC before campaign launch
- Add Smartlead or Instantly handoff after human approval and verified deliverability readiness
- Revisit multi-company-per-run batching after the one-company path is production-stable
- Delete disabled legacy Airtable nodes after the cleaner decision-board architecture remains stable across more runs

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
- Maintainable workflow structure over unnecessary parallelism during development

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

The project combines webhook-triggered automation, optional prospect discovery, public-web extraction, AI-assisted classification, deterministic scoring, retryable operational state, structured decision outputs, and review-first outreach preparation.

The goal is to help identify which organizations and brands are actually worth pursuing, reviewing, monitoring, or rejecting for partnership outreach.
