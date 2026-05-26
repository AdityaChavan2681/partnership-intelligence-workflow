# Partnership Intelligence Pipeline

Brand prospect assessment, sponsor-fit scoring, decision support, and review-first outreach preparation for sports partnership teams.

> Status: Active development. This repository currently documents the project architecture, workflow behavior, screenshots, and implementation notes. Credentials, API keys, database connection strings, local tunnel URLs, and environment-specific configuration are intentionally excluded.

> Latest verified run: A Zapier-triggered/local n8n pipeline execution completed successfully with `max_companies_per_run = 5`, using a 60-second cooldown after the first 3 companies. The run processed Prospect A, Prospect B, Prospect C, Prospect D, and Prospect E once in the same execution; saved one publishable company assessment; held four fallback-only assessments for manual review; and marked the pipeline run complete with MongoDB-backed final counts. The workflow now re-reads selected `brand_assessment_jobs` records before completion so `processed_job_count`, `completed_job_count`, `publishable_job_count`, `review_required_job_count`, `quality_gated_job_count`, and `retryable_job_count` reflect the actual saved job statuses. Review-only quality gates are separated from true retryable technical failures.

This project uses n8n to evaluate companies as potential sponsors, partners, vendors, media partners, facility partners, program partners, or activation partners for a target sports market. The current workflow is tuned for the pickleball ecosystem, but the intake fields and scoring model are designed to support other sports, events, venues, leagues, and commercial partnership categories.

The system is designed to answer:

```text
Given a target sports market and an outside company,
is this company worth pursuing, reviewing, monitoring, or rejecting as a partnership prospect?
```

---

## How to Review This Project

If you are reviewing this project quickly, start with these sections:

- **Overview**: what problem the pipeline solves and why the workflow exists
- **Product Flow**: the end-to-end user and system journey
- **Decision Layer**: how raw website evidence becomes a pursue/review/monitor/reject recommendation
- **Reliability Model**: how the workflow handles bad pages, provider errors, retries, and run locks
- **Verified Output Example**: the latest successful 5-company batch run and final MongoDB-backed counts
- **Portfolio Screenshots**: workflow, Airtable review output, and Zapier trigger setup

The strongest product signal is not that the workflow can scrape pages. It is that the system turns noisy public-web evidence into reviewable partnership decisions, keeps operational state in MongoDB, prevents weak or failed assessments from becoming outreach, and gives a human reviewer a clear next step.

---

## Product Flow

```text
Company intake
-> queue and dedupe jobs
-> inspect public website evidence
-> classify page-level partnership signals
-> score and aggregate company fit
-> produce a review decision
-> save the decision board record
-> gate outreach unless the decision is strong enough
-> update run status from MongoDB job state
```

This keeps the workflow focused on decision support before outreach. A company can complete assessment without automatically receiving an outreach draft; only publishable `pursue` decisions move into outreach preparation.

---

## Why This Matters

Partnership teams often waste time on prospects that look attractive by brand recognition but have weak evidence for the actual sports market. This workflow separates direct fits, adjacent fits, middle-confidence vendors, weak prospects, and review-needed assessments before outreach happens.

The practical value is a repeatable review system:

- stronger prioritization before manual outreach
- evidence-backed reasoning instead of gut feel
- controlled retry handling for failed or thin assessments
- a clear split between operational state and human review outputs
- safer outreach gating so uncertain records do not become prospect-facing messages

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
- Structured decision reason codes and previous-run change tracking
- Validated controlled one-to-five company batch mode for queue processing
- Airtable decision outputs and gated outreach review outputs
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
-> prepare prospect review decision with reason codes
-> compare against the latest previous completed assessment
-> classify decision/score change status
-> save company assessment to MongoDB
-> save decision record to Airtable
-> generate company-context outreach draft
-> save outreach draft to MongoDB and Airtable
-> create contact placeholder/review records
-> send owner-review email through SMTP/Gmail when enabled
-> update MongoDB job status
-> after the loop finishes, re-read selected MongoDB job records
-> mark pipeline run complete with database-backed batch counts
```

---

## Workflow State

Current workflow snapshot:

- 116 n8n nodes including disabled legacy/reference nodes
- Active Zapier-compatible webhook path: `partnership-pipeline-run`
- MongoDB queue collection: `brand_assessment_jobs`
- Page evidence collection: `brand_page_assessments`
- Company assessment collection: `brand_assessments`
- Outreach draft collection: `outreach_drafts`
- Pipeline status collections: `pipeline_run_status`, `pipeline_run_events`
- Current model routing: `models/gemini-3.1-flash-lite`
- Owner-review email sender used in testing: `test-sender@example.com`
- Default stale lock timeout for development/manual runs: 10 minutes
- Manual/dev pipeline run size: 1 to 5 companies per run, capped by runtime input
- Page classification uses one looped extractor lane instead of duplicated parallel batch branches
- Website page discovery preserves page labels such as `homepage`, `about`, `product_category`, `partners_or_sponsorship`, `events_or_athletes`, and `blog_or_news`
- Decision outputs use `decision_v2` reason-code logic plus previous-assessment change tracking
- Optional batch cooldown can pause after a configured number of processed companies
- Final batch completion counts are computed from fresh MongoDB job records after loop completion

The workflow supports a small controlled batch per manual or Zapier-triggered development run. The default remains one company, while `max_companies_per_run`, `company_batch_size`, or related queue-limit fields can raise the cap up to five companies. A 5-company batch run has been validated end to end with corrected page labels, brand-level de-dupe, quality-gated manual-review outcomes, a 60-second post-third-company cooldown, outreach gating, and MongoDB-backed final completion counts.

---

## Webhook Run Input

Webhook clients such as Zapier can send fields like this:

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
  "max_companies_per_run": 5,
  "cooldown_after_company_count": 3,
  "batch_cooldown_seconds": 60,
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
- `max_companies_per_run` controls the selected assessment queue batch size and is capped at 5 for stable manual and Zapier-driven development runs.
- `cooldown_after_company_count` and `batch_cooldown_seconds` can pause a batch after a configured number of processed companies to reduce pressure on model/provider limits.
- `pipeline_stale_after_minutes` can override the default lock timeout, capped between 1 and 90 minutes.
- `manual_sender_email` and `owner_review_email_to` are runtime-configurable and should use placeholders in public documentation.
- Prospect-facing sends are not automatic. The current delivery mode is review-first.

---

## Manual Test Prospect Mode

For demos and debugging, the webhook can bypass discovery and queue a specific company directly.

If these fields are present:

- `manual_test_company_name`
- `manual_test_root_domain`

the workflow creates or updates a `brand_assessment_jobs` record with `source_platform = manual_test`, skips Apollo discovery for that run, and assesses the provided company.

For controlled batch testing, the webhook can also send `manual_test_companies` as a JSON array. The workflow dedupes by domain, caps the manual batch at five companies, queues each company, and restricts the assessment query to the just-seeded manual source IDs.

Optional fields:

- `manual_test_priority`
- `manual_test_partner_group`
- `manual_test_relationship_context`
- `manual_test_company_limit`

Single-company example:

```json
{
  "manual_test_company_name": "Prospect A",
  "manual_test_root_domain": "https://prospect-a.example",
  "manual_test_priority": "urgent",
  "manual_test_partner_group": "vendor_partner",
  "manual_test_relationship_context": "Manual middle-fit test prospect for partnership decisioning"
}
```

Batch example:

```json
{
  "manual_test_companies": [
    {
      "name": "Prospect Alpha",
      "root_domain": "https://prospect-alpha.example",
      "priority": "urgent"
    },
    {
      "name": "Prospect A",
      "root_domain": "https://prospect-a.example",
      "partner_group": "vendor_partner"
    },
    {
      "name": "Prospect C",
      "root_domain": "https://prospect-c.example"
    }
  ],
  "max_companies_per_run": 5,
  "cooldown_after_company_count": 3,
  "batch_cooldown_seconds": 60
}
```

This mode is useful for validating decision behavior across strong, middle, and weak prospects without manually editing MongoDB. If the manual test fields are absent, the workflow continues through the normal Apollo/Mongo queue path.

---

## Decision Layer

After brand-level aggregation, the workflow prepares a prospect review decision.

Decision outputs include:

- `review_decision`: `pursue`, `review`, `monitor`, or `reject`
- `decision_confidence`: `high`, `medium`, or `low`
- `decision_reason_codes`
- `decision_reason_summary`
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

Reason codes make the decision auditable without reading every source page. Examples include:

```text
strong_score
strong_page_evidence
high_fit_pages_found
direct_product_or_gear_fit
facility_or_venue_fit
content_or_media_fit
vendor_support_only
thin_evidence
no_high_fit_pages
no_strong_evidence_pages
score_below_threshold
domain_quality_risk
assessment_needs_retry_or_verification
```

These fields are stored in MongoDB and sent to the `prospect_decisions` Airtable table so a reviewer can quickly see why a company was pursued, monitored, or rejected.

---

## Decision Change Tracking

After each prospect review decision is prepared, the workflow looks up the latest previous completed assessment for the same `brand_key`.

The comparison adds fields such as:

- `previous_review_decision`
- `previous_brand_fit_score`
- `score_delta`
- `score_delta_abs`
- `decision_changed`
- `decision_change_status`
- `decision_change_reason`
- `previous_assessment_run_id`
- `previous_assessment_date`
- `previous_decision_reason_summary`
- `decision_change_version`

Change statuses:

```text
new_assessment          -> no previous completed assessment found
stable                  -> same decision and very small score movement
minor_change            -> same decision with small score movement
material_score_change   -> same decision with meaningful score movement
decision_changed        -> review decision changed between runs
```

This gives the decision board memory. A reviewer can distinguish a newly assessed company from one whose recommendation stayed stable, improved, weakened, or changed enough to revisit.

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

- Decision: pursue/review/monitor/reject, confidence, reason codes, previous-run change status, best use case, risk, rationale, and next human step
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
- Completed runs store processed, completed, publishable, review-required, quality-gated, retryable, and failed-provider job counts.
- Final batch counts are sourced from fresh MongoDB queue records after all per-job updates are written.
- Apollo duplicate-only runs can continue into existing queued or retryable MongoDB jobs.
- Apollo provider errors, credit limits, or network errors do not block existing MongoDB queue assessment.
- Website evidence can override stale or incomplete enrichment metadata.
- Blocked, empty, failed, and soft-404 pages produce fallback records instead of disappearing silently.
- Suspicious or hijacked-looking domains can be rejected deterministically before wasting model calls.
- Model/provider failures are marked as `failed_provider` or `needs_retry` instead of being published as completed assessments.
- Quality gates prevent provider-error assessments from being treated as publishable results.
- Failed-provider assessments are blocked from outreach until a publishable assessment exists.

Status model:

```text
status = execution state, such as queued, running, completed, or failed_provider
assessment_quality_status = output quality, such as publishable, needs_review, needs_retry, or failed_provider
```

A publishable assessment can finish with `status = completed` and `assessment_quality_status = publishable`, while a fallback-heavy assessment can finish with `status = completed` and `assessment_quality_status = needs_review`. `publishable` means the output can be saved to the decision/review layer. It does not automatically mean outreach is approved; outreach still depends on `review_decision`. `needs_review` means the workflow ran correctly, but the result should only be saved as a manual-review record, not as a full publishable assessment or outreach input. `needs_retry` is reserved for technical/provider retry cases.

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
- `brand_assessments`: company-level assessment history, decision fields, reason codes, and previous-run comparison fields
- `outreach_drafts`: outreach drafts, owner-review email payloads, manual outreach status, sender-domain readiness
- `config`: runtime config, taxonomy, and crawl settings
- `pipeline_run_status`: run lock and current run status
- `pipeline_run_events`: skip/no-work/status events

The queue is intentionally lightweight. Detailed evidence, scoring, decision, and outreach data are stored downstream after assessment.

---

## Airtable Review Architecture

Airtable is used for human review, filtering, prioritization, and approval. Publishable assessments and manual-review assessment outcomes both use the `prospect_decisions` table, but only publishable `pursue` decisions continue into outreach draft generation.

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

These legacy nodes are intentionally kept for now because they are disabled, disconnected, and separated from the active canvas. They can be deleted later after the current `prospect_decisions`, `outreach_review_queue`, and `prospect_contacts` flow remains stable across more runs. Keeping them temporarily reduces rollback risk while the workflow is still in active development.

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

A recent controlled-batch test completed successfully with `max_companies_per_run = 5`, `cooldown_after_company_count = 3`, and `batch_cooldown_seconds = 60`. The status model has since been clarified so publishable decision-board results use `assessment_quality_status = publishable`, while fallback-heavy results use `assessment_quality_status = needs_review`.

Verified execution summary:

```text
Status: success
Started at: 2026-05-25T18:00:51.323Z
Stopped at: 2026-05-25T18:03:04.516Z
Measured runtime: 133.2 seconds
Top-level errors: 0
Companies selected: Prospect A, Prospect B, Prospect C, Prospect D, Prospect E
Pipeline completion status: completed
Cooldown behavior: waited once for 60 seconds after company 3
```

Before marking the pipeline complete, the workflow re-read the selected `brand_assessment_jobs` records from MongoDB. That final database check returned all five selected jobs and produced the completion counts below.

Batch completion output:

```json
{
  "status": "completed",
  "processed_job_count": 5,
  "completed_job_count": 5,
  "publishable_job_count": 1,
  "review_required_job_count": 4,
  "quality_gated_job_count": 4,
  "retryable_job_count": 0,
  "failed_provider_job_count": 0,
  "last_message": "Completed processing for 5 assessment job(s): 1 publishable, 4 held for review."
}
```

In this output, `processed_job_count = 5` and `completed_job_count = 5` mean all five selected companies reached an end state inside the same workflow execution. `publishable_job_count = 1` means one result was publishable and passed the quality gate for decision-board publishing. `review_required_job_count = 4` and `quality_gated_job_count = 4` mean four results completed but were held for manual review. `retryable_job_count = 0` means none of those four were treated as technical retry failures.

Representative decision-board output after the manual-review branch:

```json
[
  {
    "analyzed_organization": "Prospect A",
    "status": "completed",
    "assessment_quality_status": "publishable",
    "approval_status": "needs_review",
    "brand_fit_score": 92,
    "review_decision": "pursue",
    "next_human_step": "Review the top evidence pages, verify the right contact, then approve the outreach draft."
  },
  {
    "analyzed_organization": "Prospect B",
    "status": "completed",
    "assessment_quality_status": "needs_review",
    "approval_status": "needs_manual_review",
    "brand_fit_score": 65,
    "review_decision": "review",
    "manual_review_reason": "All usable pages used fallback or insufficient-evidence records; manual review is needed before publishing."
  },
  {
    "analyzed_organization": "Prospect C",
    "status": "completed",
    "assessment_quality_status": "needs_review",
    "approval_status": "needs_manual_review",
    "brand_fit_score": 60,
    "review_decision": "review",
    "manual_review_reason": "All usable pages used fallback or insufficient-evidence records; manual review is needed before publishing."
  },
  {
    "analyzed_organization": "Prospect D",
    "status": "completed",
    "assessment_quality_status": "needs_review",
    "approval_status": "needs_manual_review",
    "brand_fit_score": 4,
    "review_decision": "reject",
    "manual_review_reason": "Assessment produced a very low score and requires manual verification before any action."
  },
  {
    "analyzed_organization": "Prospect E",
    "status": "completed",
    "assessment_quality_status": "needs_review",
    "approval_status": "needs_manual_review",
    "brand_fit_score": 63,
    "review_decision": "review",
    "manual_review_reason": "All usable pages used fallback or insufficient-evidence records; manual review is needed before publishing."
  }
]
```

Representative storage, manual-review, and outreach output:

```json
{
  "brand_assessment_records_saved": 1,
  "airtable_decision_records_saved": 5,
  "manual_review_decision_records_saved": 4,
  "outreach_drafts_created": 1,
  "publishable_company": "Prospect A",
  "publishable_decision": "pursue",
  "manual_review_companies": 4
}
```

Website page discovery was also verified in this run. Page evidence now preserves the expected labels instead of incorrectly treating every discovered URL as a homepage:

```text
homepage
about
product_category
partners_or_sponsorship
events_or_athletes
blog_or_news
```

The outreach branch creates review-ready email drafts, owner-review email notifications, manual outreach status fields, and contact-role placeholders before any prospect-facing send step is considered.

`needs_review` assessments now save lightweight manual-review records to the Airtable decision board with score, quality status, source URLs, risk, and next human step. Only `pursue` decisions generate outreach drafts automatically. `review`, `monitor`, `reject`, `needs_review`, and technical `needs_retry` assessments are held back from prospect-facing outreach.

---

## Decision Spectrum Validation

Recent test runs were used to confirm that the workflow can distinguish strong, adjacent, middle, and weak prospects instead of blindly rewarding brand recognition.

| Company | Fit Pattern | Score | Decision | Outreach Draft |
|---|---:|---:|---|---|
| Direct Equipment Brand | direct pickleball product fit | 100 | pursue | yes |
| Adjacent Hydration Brand | adjacent hydration/event activation fit | 100 | pursue | yes |
| Prospect A | middle vendor/content support fit | 47 | monitor | no |
| Prospect C | weak/non-sport-specific fit | 6 | reject | no |

The weak-fit enterprise prospect test is important because it shows that a large, well-known company can still be rejected when the public evidence does not support a credible target-market partnership angle. The middle-fit vendor test is important because it shows a moderate operational/content use case can be retained for monitoring without triggering outreach.

Later middle-fit prospect reruns were also used to validate the historical comparison layer. In the latest exported run, the anonymized middle-fit prospect produced a publishable decision-board record while remaining separate from stronger `pursue` examples used for outreach validation.

---

## Portfolio Screenshots

### Workflow Overview

<img width="1672" height="403" alt="image" src="https://github.com/user-attachments/assets/d2525e2b-4767-4f31-808e-712fb7619fcc" />

### Airtable Review Output

<img width="1260" height="632" alt="image" src="https://github.com/user-attachments/assets/81200018-7df4-47dd-84d6-7e6d4bcec0f8" />

### Zapier Trigger Setup

<img width="416" height="385" alt="image" src="https://github.com/user-attachments/assets/1a27b6d0-bbf0-4aa9-a22e-0458294c1091" />

---

## Test Notes

Recent validation runs confirmed:

- a full Zapier-triggered/local pipeline run completed successfully with zero top-level errors
- the latest controlled 5-company batch run completed in 133.2 seconds, including one configured 60-second cooldown
- manual test prospect mode can queue and assess a named company directly from webhook fields
- controlled batch mode processed 5 selected companies once in one pipeline run
- page discovery now preserves differentiated page labels instead of collapsing all discovered URLs into `homepage`
- brand-level de-dupe prevented duplicate MongoDB/Airtable decision records
- final run status correctly separates completed processing from publishable, manual-review, and retryable outcomes
- manual test prospect mode skips Apollo discovery for that run and still uses the MongoDB queue
- Apollo insufficient-credit responses are detected and skipped without blocking assessment
- Apollo network/DNS/provider errors are configured to continue into the skip path
- the AI page classifier loop processes selected pages through a single reusable lane
- publishable assessments generate MongoDB assessment records and Airtable decision records; `needs_review` assessments generate Airtable manual-review decision records with score and reason; only `pursue` decisions continue into outreach drafts, contact-role records, and owner-email outputs
- decision reason codes and summaries save into the decision record
- the workflow compares each company against its latest previous completed assessment when history exists
- decision-change output has been validated across repeated company runs, including stable and updated anonymized prospect assessments
- `monitor` decisions save to `prospect_decisions` without generating outreach drafts
- `reject` decisions save to `prospect_decisions` without generating outreach drafts
- only `pursue` decisions currently generate outreach drafts and owner-review emails
- `needs_review` and technical `needs_retry` assessments are excluded from outreach draft generation while still preserving manual-review context where appropriate
- owner-review email configuration now comes from Zapier/webhook runtime fields
- the pipeline run lock starts as `running` and finishes as `completed`
- outreach drafts save to MongoDB and Airtable review tables
- completion status now re-reads selected MongoDB queue records after the loop to avoid stale n8n loop-memory counts
- batch cooldown after the third selected company was verified in a 5-company run
- prospect decision records and manual-review records save to the `prospect_decisions` Airtable table
- sender-domain inputs from Zapier flow into outreach readiness fields
- manual outreach status and follow-up fields are available for review-first sending
- failed-provider assessments are blocked from outreach until a publishable assessment exists
- a transient Gemini page-classification service-unavailable response was handled without crashing the workflow

These examples are prototype test results derived from public-web data. They do not represent official recommendations, endorsements, or affiliations with any listed organization.

---

## What This Demonstrates

- Designing stateful automation instead of one-off scraping
- Building retry-aware workflows around unreliable external services
- Combining AI classification with deterministic validation and scoring
- Turning messy public-web evidence into review-ready business outputs
- Adding decision support on top of raw scoring
- Tracking whether a decision is new, stable, materially changed, or changed outright
- Preparing outbound-ready prospect intelligence before automated sending
- Generating company-context email drafts from assessment evidence
- Sending automatic owner-review notifications while gating prospect-facing outreach
- Separating operational state in MongoDB from human review workflows in Airtable
- Refactoring duplicated AI branches into a maintainable loop without changing downstream outputs
- Keeping public documentation safe by using placeholder runtime email values

---

## Key Capabilities

- Zapier-compatible webhook intake
- Webhook-driven manual test prospect seeding
- Validated controlled small-batch queue processing with optional cooldown
- Optional Apollo company discovery
- Optional Clay enrichment handoff
- MongoDB-backed queueing and retry state
- One-to-five company manual pipeline runs for stable development execution
- Pipeline run lock with stale timeout and MongoDB-backed completion status
- Company/root-domain dedupe
- Public-web page discovery and fetch validation
- Corrected page-role labeling for discovered website URLs
- Blocked, empty, soft-404, and suspicious-domain fallback handling
- Multi-page company context extraction
- Looped AI-assisted page classification
- Deterministic scoring and validation guardrails
- Provider-failure recovery and retryable job states
- Page-level evidence storage
- Company-level sponsor-fit aggregation
- Short-term and long-term partnership potential scoring
- Prospect review decision layer
- Decision reason-code explainability
- Previous-assessment decision change tracking
- Airtable decision board output
- Company-context outreach draft generation
- Automatic owner-review email notification for generated outreach drafts
- Manual outreach tracking and sender-domain readiness fields

---

## Roadmap

- Keep validating the single-loop page classifier across broader company categories
- Add automated retry scheduling for true technical retry states such as `failed_provider` and `needs_retry` jobs
- Add stronger run-level observability for current node, provider failures, request usage, and retry reasons
- Expand historical analytics beyond latest-previous comparison when enough assessment history exists
- Further simplify MongoDB queue seed records into a minimal input schema
- Improve automated discovery of company websites when only a company name is provided
- Reduce AI calls with tighter pre-AI page selection and text limits
- Add person/contact enrichment only after a prospect reaches a review threshold
- Add automated outbound domain-readiness checks for SPF, DKIM, and DMARC before campaign launch
- Add Smartlead or Instantly handoff after human approval and verified deliverability readiness
- Continue validating controlled multi-company batch runs across stronger and weaker prospect mixes
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
- Partnership target discovery
- Outreach prioritization
- Sponsorship category research
- Manual outreach preparation
- Historical monitoring of public commercial signals
- Sport-level partnership intelligence

