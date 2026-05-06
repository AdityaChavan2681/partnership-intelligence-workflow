# Partnership Intelligence Pipeline

  ### Brand Prospect Assessment, Sponsor-Fit Scoring, and Public-Web Partnership Intelligence

  A public-web intelligence pipeline for evaluating outside brands and companies as potential sponsors or partners for a target sports property.

  The current workflow starts from a brand root domain, discovers relevant internal pages, extracts public evidence, classifies sponsor-fit signals, scores each page, and aggregates the strongest evidence into a company-level brand assessment.

  The system is designed to answer:

  ```text
  Given a target sports property and an outside company,
  is this company a credible sponsor or partner prospect?
  ```

  ---

  ## Overview

  Partnership teams often evaluate sponsor prospects from scattered public signals: homepages, product pages, partner pages, athlete pages, event pages, news posts, and contact pages. These signals are useful, but they are noisy and
  difficult to compare manually.

  This project converts that public evidence into structured sponsor-prospect intelligence that can be scored, reviewed, stored, and monitored over time.

  The pipeline uses a hybrid approach:

  - Homepage-based page discovery
  - Clean text extraction from public webpages
  - Source metadata inference
  - Pre-AI heuristics
  - Multi-page company context extraction before page-level classification
  - AI-assisted page classification
  - Deterministic validation and fallback handling
  - Page-level brand-fit scoring
  - Company-level sponsor-fit aggregation
  - MongoDB storage for evidence and history
  - Airtable output for review-ready assessments

  ---

  ## Current Workflow: Brand Prospect Assessor

  The Brand Prospect Assessor evaluates an outside brand or company as a potential sponsor or partner for a target sports property, and can also generate review-ready outreach drafts from the strongest results.

  Assessment jobs are loaded from a MongoDB-backed `brand_assessment_jobs` queue. Jobs can come from manual input or from the Apollo prospect intake branch, which discovers companies, normalizes them into queued jobs, and stores them in MongoDB for assessment.

  During testing, the workflow can target a specific queued organization by adding `analyzed_organization` to the MongoDB query; in normal operation, it can select active queued jobs from the collection.

  Starting from a brand root domain, it discovers and evaluates relevant pages such as:

  - Homepage
  - About page
  - Product/category pages
  - Partner/sponsorship pages
  - Blog/news pages
  - Events/athlete pages
  - Contact pages as supporting context only

  Each page is classified and scored individually. The workflow then aggregates the strongest page-level evidence into one company-level sponsor-fit assessment.

  Before page-level AI classification, the workflow also builds a multi-page company context summary from selected pages. This helps identify what the analyzed organization primarily does across its public website, while still keeping final scoring grounded in each page’s own evidence.

  ### Page Discovery and Batch Processing

  The workflow starts from a single brand root domain and fetches the homepage. It then discovers internal links from that homepage, filters out low-value paths such as cart, checkout, login, privacy, terms, shipping, search, sale, and static asset URLs, and classifies the remaining pages by role.

  The current crawl configuration evaluates up to 10 selected pages across roles such as homepage, about, contact, product/category, blog/news, partner/sponsorship, and events/athlete pages.

  Selected pages are sorted by AI priority, assigned into batches of two pages, and routed across five AI classification branches. The branches share one Gemini chat model connection and use staggered waits plus retry handling for more stable structured classification under provider rate limits.

  After classification, batch outputs are normalized, merged, validated, scored at the page level, stored in MongoDB, and aggregated into one company-level brand assessment, with Airtable review output for assessments and outreach drafts.

  ### Main Outputs

  - `brand_fit_score`
  - `brand_fit_conclusion`
  - `recommended_action`
  - `company_category`
  - `business_model`
  - `recommended_offer`
  - `primary_partnership_angle`
  - `primary_evidence_summary`
  - `source_urls`
  - `page_count`
  - `usable_page_count`
  - `brand_result_signature`
  - `partner_group`
  - `evidence_page_count`
  - `strong_evidence_page_count`
  - `high_fit_page_count`
  - `top_evidence_pages`
  - `brand_key`
  - `outreach_stage`
  - `approval_status`
  - `contact_role_targets`
  - `outreach_subject`
  - `outreach_body`

  ---

  ## Architecture

  ```text
  MongoDB queued assessment job
  → Select active queued brand/company
  → Apollo prospect intake or manual job selection
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
     └─ Airtable outreach review queue
  → MongoDB job-status update
  ```

  ---

  ## Pipeline Snapshot

  The current n8n workflow includes Apollo prospect intake, page discovery, text extraction, pre-AI heuristics, multi-page company context extraction with a Gemini model and wait timer, AI classification batches, validation, page-level scoring, company-level aggregation, MongoDB storage, and Airtable review outputs for both brand assessments and outreach drafts.
  
  <img width="1505" height="395" alt="image" src="https://github.com/user-attachments/assets/4ca2f18e-ea4c-4c64-a3c8-75745b083a11" />

  ---

  ## Current Reliability Features

  The workflow includes several deterministic safeguards around public-web extraction and AI classification:

  - Homepage and discovered-page HTTP requests use browser-like headers such as `User-Agent`, `Accept`, `Accept-Language`, and `Cache-Control` to reduce avoidable 403/Cloudflare blocks.
  - Blocked, failed, empty, and 404-like pages are detected before AI classification.
  - Blocked pages can produce an insufficient-evidence fallback record instead of silently disappearing.
  - Homepage discovery stops early when the fetched homepage is a Cloudflare/error page.
  - Page discovery filters remove low-value utility paths, static assets, checkout/login pages, and unrelated noisy industry pages.
  - AI batch outputs are normalized with deterministic guardrails for contact pages, generic blog indexes, trade-event pages, medical case studies, adult beverages, healthy beverages, software vendors, and non-core product category pages.
  - Direct target-organization evidence is preserved for vendor/software partners that explicitly reference the target sports property.
  - Page-level and aggregate outputs preserve raw AI output, normalized fields, evidence summaries, source URLs, scoring metadata, and run signatures.
  - MongoDB job-status updates mark completed jobs with `last_run_at`, `last_run_id`, and cleared errors.
  - Page validation catches both hard 404s and soft 404 pages such as "No Results Found" pages before AI classification.
  - Blocked, failed, empty, hard-404, and soft-404 pages produce insufficient-evidence fallback records with zero page score.
  - Runtime and taxonomy configuration are loaded from MongoDB config documents, with validation to prevent missing or duplicate active config docs.
  - Airtable is treated as a review layer; MongoDB result storage and job-status updates do not depend on Airtable success.
  - Vendor/software guardrails distinguish generic B2B software from sports facility software such as court reservation, club management, event management, and facility operations platforms.
  - Multi-page company context extraction helps distinguish what the analyzed organization primarily does from noisy page-level keyword matches.
  - Company-level context is attached back to page records before AI batching, while page scoring remains grounded in current-page evidence.
  - Apparel and activewear guardrails prevent general lifestyle brands from being over-scored as direct pickleball fits without explicit court-sport, venue, event, athlete, or sponsorship evidence.
  - Control tests include both relevant and non-relevant companies to verify that the workflow does not force weak prospects into sponsor categories.

  ---

  ## Storage

  ### MongoDB

  MongoDB is used for structured storage and historical evidence.

  Current brand assessment storage includes:

  - `brand_page_assessments`
  - `brand_assessments`
  - `brand_assessment_jobs`
  - `outreach_drafts`
  - `config`

  ### Airtable

  Airtable is used as an human-facing review layer.

  Current Airtable review outputs use:

  - `brand_assessments`
  - `outreach_review_queue`

  The `brand_assessments` table stores company-level sponsor-fit assessments.
  The `outreach_review_queue` table stores review-ready outreach drafts.

  ---

  ## Example Input

  
  ### MongoDB Job Input

  In normal operation, the workflow reads assessment jobs from `brand_assessment_jobs`. During testing, the MongoDB query can target one organization by `analyzed_organization`.
  
  ```json
  {
    "active": true,
    "status": "queued",
    "partner_group": "brand_partner",
    "target_organization": "Example Pickleball Team",
    "target_org_type": "sports team",
    "target_market": "pickleball / sports / events",
    "analyzed_organization": "Example Pickleball Equipment Brand",
    "analyzed_org_type": "brand",
    "root_domain": "https://example.com",
    "relationship_context": "Known or potential brand partner for Example Pickleball Team",
    "config_version": "brand_assessment_v1",
    "priority": "normal",
    "created_at": "2026-05-01",
    "updated_at": "2026-05-01",
    "last_run_at": null,
    "last_run_id": null,
    "last_error": null
  }
  ```

  ---

  ## Example Output

  ### Brand Assessment Result

  ```json
  {
    "target_organization": "Example Pickleball Team",
    "analyzed_organization": "Example Pickleball Equipment Brand",
    "partner_group": "brand_partner",
    "company_category": "pickleball equipment",
    "business_model": "consumer sports equipment",
    "recommended_offer": "player gear partnership",
    "brand_fit_score": 100,
    "brand_fit_conclusion": "strong brand fit",
    "recommended_action": "prioritize outreach",
    "primary_partnership_angle": "As a manufacturer of pickleball paddles and equipment, the company is a natural fit for team sponsorship, gear supply, and co-branded activations.",
    "page_count": 9,
    "usable_page_count": 8,
    "evidence_page_count": 8,
    "aggregate_version": "brand_v1"
  }
  ```

  ---

  ## Review Output

  Company-level assessments are written to Airtable for review, and outreach drafts are written to a separate Airtable review queue for approval before any send step is added.

  ---

  ## Current Test Results

  The workflow has been tested end-to-end on several partner categories using an example pickleball team as the target sports-property context.

  Note: These examples are anonymized prototype test results derived from public-web data. They are shown as category-level examples only and do not represent official partner recommendations, endorsements, or affiliations with any
  listed organization.

  | Example Company Type | Category | Partner Group | Score | Conclusion |
  |---|---|---|---:|---|
  | Pickleball equipment brand | pickleball equipment | brand_partner | 100 | strong brand fit |
  | Pickleball travel company | pickleball travel and experiences | experience_partner | 100 | strong brand fit |
  | Sports facility software vendor | sports facility software | vendor_partner | 86 | strong brand fit |
  | Activewear/lifestyle apparel brand | sports apparel | brand_partner | 61 | moderate brand fit |
  | General productivity software control | software | brand_partner | 7 | weak brand fit |
  | Insurance/risk services firm | insurance and risk management | vendor_partner | 47 | moderate brand fit |
  | Hydration beverage brand | hydration beverage | brand_partner | 41 | moderate brand fit |
  | Coffee brand | beverage / coffee | brand_partner | 39 | weak brand fit |
  | Adult beverage brand | adult beverage | brand_partner | 30 | weak brand fit |

  These tests validate that direct pickleball and experience partners rank highest, facility-software vendors can be recognized as vendor partners, Apollo-discovered prospects can be normalized into the same assessment queue, and outreach drafts can be reviewed separately in Airtable before any send step is added.
  
  ---

  ## Key Capabilities

  - Multi-page brand/company assessment
  - Multi-page company context extraction before page-level AI classification
  - Homepage-based internal page discovery
  - Source metadata inference
  - Clean text extraction from public webpages
  - AI classification with structured fields
  - Deterministic validation and fallback handling
  - Page-level fetch validation and failed-page suppression
  - AI-output normalization guardrails
  - Historical run-level assessment keys
  - Page-level brand-fit scoring
  - Company-level sponsor-fit aggregation
  - Airtable output for review-ready assessments and outreach review
  - MongoDB storage for page-level evidence and aggregate assessments
  - Early scan-memory and change-detection support

  ---

  ## Prototype History

  This project began as a sports-property signal scanner for an example pickleball team, including pages such as brand partners, past events, and latest news.

  That prototype established the core system patterns:

  - Source metadata inference
  - Clean text extraction
  - AI-assisted classification
  - Validation and fallback handling
  - Scoring logic
  - Early scan-memory signatures

  The current workflow builds on those patterns but generalizes the system toward reusable brand prospect assessment across outside companies and sponsor categories.

  ---
  
  ## Roadmap

  - Add material-change detection for repeated brand scans
  - Reduce AI token usage with tighter page selection and text limits
  - Batch assessment across multiple brands
  - Rank brands across sponsor categories
  - Add outreach-ready summaries
  - Add more Airtable review views for priority queues and follow-up tracking
  - Add stronger upsert/dedupe behavior for MongoDB and Airtable records
  - Add seed URL support for known high-value pages such as sports insurance, sponsorship, or partner pages
  - Add browser-rendered fetch fallback for sites that require JavaScript or advanced bot protection
  - Add category-specific page summary guardrails for apparel, recovery, accessories, and service pages
  - Add human-readable top-evidence summaries for Airtable
  - Move more hardcoded category, guardrail, and controlled-value logic into MongoDB taxonomy/runtime config, including reusable taxonomy matching before AI classification
  - Add Airtable outreach approval status tracking
  - Add Smartlead or Instantly send-step integration after approval

  ---

  ## Design Principles

  - Multi-source evidence over single-page assumptions
  - Deterministic validation over blind AI trust
  - Page-level evidence before company-level conclusions
  - MongoDB for memory and depth
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

  ## Why This Matters

  This is more than a scraping workflow.

  The project combines public-web extraction, AI-assisted classification, deterministic scoring, scan memory, MongoDB storage, and a two-stage review flow in Airtable for both company assessments and outreach drafts.

  The goal is to help identify which organizations and brands are actually worth reviewing, ranking, and prioritizing for partnership outreach.

