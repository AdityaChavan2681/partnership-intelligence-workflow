# Partnership Intelligence Pipeline

  ### Brand Prospect Assessment, Sponsor-Fit Scoring, and Public-Web Partnership Intelligence

  A public-web intelligence pipeline for evaluating companies as potential sponsors or partners within a target sports market.

  The current workflow focuses on the pickleball ecosystem. It uses Apollo for prospect discovery, optional Clay handoff for enrichment, n8n for orchestration, MongoDB for queueing and evidence history, Gemini for structured classification, and Airtable for review-ready outputs.

  The system is designed to answer:

  ```text
  Given a target sport, league, event, venue, or sports property and an outside company,
  is this company a credible sponsor or partner prospect?
  ```

  ---

  ## Overview

  Partnership teams often evaluate sponsor prospects from scattered public signals: homepages, product pages, partner pages, athlete pages, event pages, news posts, media pages, and contact pages. These signals are useful, but they are noisy and difficult to compare manually.

  This project converts that public evidence into structured sponsor-prospect intelligence that can be scored, reviewed, stored, retried, and monitored over time.

  The pipeline combines:

  - Apollo prospect discovery
  - Clay company enrichment handoff
  - n8n workflow orchestration
  - MongoDB queue, evidence, retry, and history storage
  - Public website discovery and text extraction
  - Gemini-assisted company and page classification
  - Deterministic scoring and quality gates
  - Airtable review outputs for assessments, rankings, contacts, and outreach

  ---

  ## Architecture

  ```text
  Zapier or HTTP webhook trigger
  → n8n pipeline run webhook
  → Apollo prospect intake
  → Normalize discovered companies into lightweight candidate jobs
  → Dedupe against existing MongoDB jobs by source ID and root domain
     ├─ If new prospects exist: queue them in MongoDB and send optional Clay enrichment handoff
     └─ If no new prospects exist: continue to queued/retryable assessment jobs
  → Select queued, needs-retry, or provider-failed assessment job
  → Attach runtime and taxonomy config
  → Brand root domain
  → Homepage fetch with browser-like headers
  → Attach homepage HTML and fetch metadata
  → Internal link discovery with blocked-homepage detection
  → Relevant page filtering and role classification
  → Fetch selected pages with browser-like headers, up to 10 total
  → Page-level fetch validation for HTTP errors, hard/soft 404s, empty pages, and Cloudflare blocks
  → Clean text extraction with evidence-snippet preservation
  → Require usable page text
     ├─ If text exists: pre-AI heuristics, taxonomy hints, and priority sorting
     └─ If text is missing: blocked/empty-page fallback assessment
  → Build multi-page company context input from selected usable pages
  → Extract company-level context from multiple webpages
  → Attach company context back to page-level records
  → Batch assignment, 2 pages per AI branch
  → Gemini company-context and page-classification extraction with retry/backoff
     ├─ If AI succeeds: continue to page validation and scoring
     └─ If provider fails: mark job as failed_provider / needs_retry instead of publishing
  → Five AI classification branches
  → Deterministic AI-output normalization
  → Merge AI-classified pages and fallback page assessments
  → Page classification guardrails
  → Validate page classifications
  → Fallback handling for weak or invalid AI output
  → Assemble page assessment record
  → Page-level sponsor-fit scoring
  → Compute assessment keys and signatures
  → MongoDB page evidence storage
→ Brand-level aggregation
       ├─ MongoDB company-level assessment storage
       ├─ Airtable review output for brand assessments
       ├─ MongoDB outreach draft storage
       ├─ Airtable outreach review queue
       └─ Airtable prospect contact-role staging
    → MongoDB job-status update
  ```

  ---

  ## Data Flow

  The pipeline starts from a webhook-triggered run request, which can be sent by Zapier, another automation tool, or a direct HTTP POST.

  ```text
  Apollo company discovery
  → optional Clay enrichment/reference data
  → n8n normalization and dedupe
  → MongoDB operational queue
  → public website discovery and evidence extraction
  → Gemini-assisted classification
  → deterministic scoring and quality gates
  → MongoDB evidence/history storage
  → Airtable review outputs

  - Apollo supplies company discovery input.
  - Clay can receive optional company enrichment handoffs from n8n.
  - Clay enrichment is not required for assessment execution.
  - MongoDB stores the operational queue, dedupe state, retry state, evidence, and assessment history.
  - Airtable stores human-facing review outputs.
  ```

  ---

  ## Current Workflow

  The Brand Prospect Assessor evaluates an outside company as a potential sponsor or partner for a target sports market.

  Prospects enter the workflow through a webhook-triggered Apollo discovery run or a compatible normalized input record. n8n normalizes those records into assessment jobs, dedupes them against existing prospects, and stores operational state in MongoDB so jobs can be retried, enriched, assessed, and reviewed over time.

  Starting from a company root domain, the workflow discovers and evaluates relevant pages such as:

  - Homepage
  - About page
  - Product/category pages
  - Partner/sponsorship pages
  - Blog/news/media pages
  - Events/athlete pages
  - Contact pages as supporting context only

  Each page is classified and scored individually. The workflow then aggregates the strongest page-level evidence into one company-level sponsor-fit assessment.

  Before page-level AI classification, the workflow builds a multi-page company context summary. This helps identify what the analyzed organization primarily does across its website, while keeping final scoring grounded in each page’s own evidence.

  ---

  ### Main Outputs

  The workflow produces five review layers:

  - **Assessment:** score, conclusion, recommended action, category, business model, offer type
  - **Evidence:** source URLs, top evidence pages, evidence summaries, page counts
  - **Reliability:** quality status, retry flag, provider-error count, fallback/blocked-page counts
  - **Ranking:** rank tier, reliability level, evidence strength, recommended action, ranking rationale
  - **Action:** contact-role targets and outreach draft fields

  ---

  ## Pipeline Snapshot

  The current n8n workflow includes Zapier-compatible webhook intake, Apollo prospect discovery, optional Clay handoff, page discovery, text extraction, pre-AI heuristics, multi-page company context extraction, AI classification batches, validation, scoring, aggregation, MongoDB storage, Airtable review outputs, and prospect contact-role staging.
  
  <img width="1501" height="346" alt="image" src="https://github.com/user-attachments/assets/4409f55c-feb3-417e-a6ca-cd347c26a5bf" />

  ---

  ## Recent Workflow Updates

  - Replaced manual/scheduled execution with a Zapier-compatible webhook trigger, so external automation tools can start the pipeline through an HTTP POST.
  - Added webhook-controlled Apollo search inputs, including keywords, location, page count, page size, partner group, and target context.
  - Removed the separate Clay return webhook branch to keep the workflow linear; Clay now remains an optional outbound enrichment handoff.
  - Added a `Needs Clay Enrichment?` gate so already-enriched records skip Clay while unenriched records can still be sent when a Clay webhook URL is configured.
  - Improved duplicate-handling behavior so Apollo duplicate-only runs can continue into existing queued or retryable assessment jobs instead of stopping.
  - Added provider-failure quality gates so transient Gemini failures are marked as retryable instead of being published as completed assessments.
  - Added latest-state Airtable upserts and append-only assessment history support for easier comparison across repeated runs.
  - Added richer partnership prospect ranking fields to explain why a prospect ranked where it did, including evidence strength, signal flags, source count, company size, location, risk flags, and recommended action.
  
  ---

  ## Reliability Model

  The workflow is designed to avoid publishing weak or failed assessments as successful results.

  - Apollo duplicate-only runs fall through to existing queued or retryable jobs.
  - Website evidence can override stale or misleading enrichment metadata.
  - Blocked, empty, failed, and soft-404 pages produce fallback records instead of disappearing.
  - Gemini calls use retry/backoff and staggered batch timing.
  - Provider failures are marked as failed_provider or needs_retry, not completed.
  - Quality gates prevent provider-error assessments from being published to review outputs.
  - Airtable is treated as a review layer; MongoDB remains the operational store.

  ---

  ## Storage

  ### MongoDB

  MongoDB is used for operational state, evidence storage, and historical assessment records.

  Current MongoDB collections include:

  - brand_assessment_jobs
  - brand_page_assessments
  - brand_assessments
  - outreach_drafts
  - config

  brand_assessment_jobs is designed to act as a lightweight operational queue. Rich Apollo, Clay, scoring, ranking, and outreach fields are preserved in assessment, enrichment, ranking, and review outputs rather than being required as initial queue input.

  ### Airtable

  Airtable is used as a human-facing review layer.

  Current Airtable review outputs use:

  - `company_enrichment`
  - `brand_assessments`
  - `partnership_prospect_rankings`
  - `prospect_contacts`
  - `outreach_review_queue`

  The `company_enrichment` table stores Clay/Apollo-enriched company reference records.
  The `brand_assessments` table stores company-level sponsor-fit assessments.
  The `partnership_prospect_rankings` table stores ranked, review-ready prospect prioritization records with score, fit, evidence strength, reliability level, and ranking rationale.
  The `prospect_contacts` table stores recommended contact-role placeholders and future person-level enrichment records.
  The `outreach_review_queue` table stores review-ready outreach drafts.

  ---

  ## Integrations

  ### Apollo

  Apollo is used for company discovery. It supplies prospect companies, domains, industry metadata, location data, employee estimates, LinkedIn URLs, and keyword signals.

  ### Clay

  Clay is used as an optional enrichment/reference-data handoff. n8n can send company payloads to Clay when enrichment is needed, while the main assessment continues without requiring a Clay callback.

  ### MongoDB

  MongoDB stores dedupe state, queue state, retry status, page-level evidence, company-level assessment history, and outreach draft records.

  ### Airtable

  Airtable stores review-ready outputs for human sorting, filtering, prioritization, and approval.

  ---

  
  ## Webhook Run Input

  The workflow can be triggered by Zapier, another automation tool, or a direct HTTP POST.
  
  ```json
  {
    "keywords": "pickleball",
    "organization_locations": "United States",
    "per_page": 5,
    "max_pages": 1,
    "partner_group": "brand_partner",
    "target_organization": "Pickleball",
    "target_org_type": "sport",
    "target_market": "pickleball / sports / events",
    "analyzed_org_type": "brand"
  }
  ```

  ## Prospect Input Contract

  n8n converts this webhook run request into normalized MongoDB assessment jobs after Apollo discovery and dedupe. Apollo and optional enrichment sources can add richer metadata, but the assessment only requires a company name and root domain once a job is queued.

  ---

  ## Review Output Contract

  The main output is a review-ready company-level partnership assessment written to MongoDB and Airtable.

  ```json
  {
    "target_organization": "Pickleball",
    "analyzed_organization": "The Dink Pickleball",
    "partner_group": "program_partner",
    "company_category": "pickleball media",
    "business_model": "b2b media and content",
    "recommended_offer": "content collaboration",
    "brand_fit_score": 100,
    "brand_fit_conclusion": "strong brand fit",
    "recommended_action": "prioritize outreach",
    "primary_partnership_angle": "A direct pickleball media partner for content collaboration, event promotion, and audience engagement.",
    "primary_evidence_summary": "Public pages show dedicated pickleball media coverage, gear reviews, and community content.",
    "evidence_page_count": 3,
    "assessment_quality_status": "complete"
  }
  ```

  A separate `partnership_prospect_rankings` output translates the assessment into a prioritization record for sorting prospects by fit, reliability, evidence strength, and recommended next action.
  
  ---

  ## Airtable Review Tables

  Company-level assessments are written to Airtable for review, outreach drafts are written to a separate Airtable review queue, ranked prospects are written to `partnership_prospect_rankings`, and recommended contact roles are staged in `prospect_contacts` before any send step is added.

  ---

  ## Current Test Results

  The workflow has been tested end-to-end on several partner categories using pickleball as the target sport and market context.

  | Example Company Type | Category | Partner Group | Score | Conclusion |
  |---|---|---|---:|---|
  | Pickleball equipment brand | pickleball equipment | brand_partner | 100 | strong brand fit |
  | Pickleball media company | pickleball media | program_partner | 100 | strong brand fit |
  | Pickleball travel company | pickleball travel and experiences | experience_partner | 100 | strong brand fit |
  | Sports facility software vendor | sports facility software | vendor_partner | 86 | strong brand fit |
  | Activewear/lifestyle apparel brand | sports apparel | brand_partner | 61 | moderate brand fit |
  | General productivity software control | software | brand_partner | 7 | weak brand fit |

  A recent control run confirmed that when enrichment metadata suggested a pickleball facility but the live website showed unrelated or spam-like content, the workflow produced a zero-score weak-fit result instead of forcing a false positive.

  These examples are prototype test results derived from public-web data. They do not represent official recommendations, endorsements, or affiliations with any listed organization.
  
  ---

  ## Key Capabilities

  - Multi-page company assessment
  - Apollo prospect intake
  - Optional Clay enrichment handoff
  - Zapier-compatible webhook intake
  - Public-web evidence extraction
  - Multi-page company context extraction
  - AI-assisted page classification
  - Deterministic scoring and quality gates
  - Provider-failure recovery and retryable job states
  - Page-level evidence storage
  - Company-level sponsor-fit aggregation
  - Airtable review outputs for assessments, contacts, and outreach
  - Prospect contact-role staging
  - Early scan-memory and change-detection support

  ---
  
  ## Roadmap

  - Further simplify MongoDB input records into a minimal queue schema
  - Add automated retry scheduling for failed_provider and needs_retry jobs
  - Add material-change detection for repeated brand scans
  - Reduce AI token usage with tighter page selection and text limits
  - Batch assessment across multiple companies
  - Rank prospects across sponsor categories
  - Add optional Apollo organization enrichment before assessment
  - Add person/contact enrichment only after a prospect reaches a review threshold
  - Add stronger upsert/dedupe behavior for MongoDB and Airtable records
  - Move more category, guardrail, and controlled-value logic into MongoDB taxonomy/runtime config
  - Add Smartlead or Instantly send-step integration after approval

  ---

  ## Design Principles

  - Multi-source evidence over single-page assumptions
  - Deterministic validation over blind AI trust
  - Page-level evidence before company-level conclusions
  - Enrichment metadata as context, not final truth
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

  The project combines webhook-triggered automation, prospect discovery, optional company enrichment, public-web extraction, AI-assisted classification, deterministic scoring, retryable operational state, and structured review outputs.

  The goal is to help identify which organizations and brands are actually worth reviewing, ranking, and prioritizing for partnership outreach.

