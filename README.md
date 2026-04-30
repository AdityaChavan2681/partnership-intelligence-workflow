# Partnership Intelligence Pipeline

  ### Brand Prospect Assessment, Sponsor-Fit Scoring, and Public-Web Partnership Intelligence

  A public-web intelligence pipeline for evaluating outside brands and companies as potential sponsors or partners for a target sports property.

  The current workflow starts from a brand root domain, discovers relevant internal pages, extracts public evidence, classifies sponsor-fit signals, scores each page, and aggregates the strongest evidence into a company-level brand
  assessment.

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
  - AI-assisted page classification
  - Deterministic validation and fallback handling
  - Page-level brand-fit scoring
  - Company-level sponsor-fit aggregation
  - MongoDB storage for evidence and history
  - Airtable output for review-ready assessments

  ---

  ## Current Workflow: Brand Prospect Assessor

  The Brand Prospect Assessor evaluates an outside brand or company as a potential sponsor or partner for a target sports property.

  Assessment jobs are loaded from a MongoDB-backed `brand_assessment_jobs` queue. Each job defines the target sports property, partner group, analyzed organization, root domain, relationship context, config version, and queue status.
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

  ### Page Discovery and Batch Processing

  The workflow starts from a single brand root domain and fetches the homepage. It then discovers internal links from that homepage, filters out low-value paths such as cart, checkout, login, privacy, terms, shipping, search, sale, and
  static asset URLs, and classifies the remaining pages by role.

  The current crawl configuration evaluates up to 10 selected pages across roles such as homepage, about, contact, product/category, blog/news, partner/sponsorship, and events/athlete pages.

  Selected pages are sorted by AI priority, assigned into batches of two pages, and routed across five AI classification branches. The branches share one Gemini chat model connection and use staggered waits plus retry handling for more stable structured classification under provider rate limits.

  After classification, batch outputs are normalized, merged, validated, scored at the page level, stored in MongoDB, and aggregated into one company-level brand assessment for MongoDB and Airtable.

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

  ---

  ## Architecture

  ```text
  MongoDB queued assessment job
  → Select active queued brand/company
  → Brand root domain
  → Homepage fetch
  → Internal link discovery
  → Relevant page filtering and role classification
  → Fetch selected pages, up to 10 total
  → Page-level fetch validation and failed-page suppression
  → Clean text extraction
  → Source metadata inference
  → Pre-AI heuristics and priority sorting
  → Batch assignment, 2 pages per AI branch
  → Five AI classification branches
  → Deterministic AI-output normalization and merge batch outputs
  → Validate page classifications
  → Fallback handling for weak or invalid AI output
  → Page-level sponsor-fit scoring
  → MongoDB page evidence storage
  → Brand-level aggregation
  → MongoDB company-level assessment storage
  → Airtable review output
  → Optional MongoDB job-status update
  ```

  ---

  ## Pipeline Snapshot

  The current n8n workflow includes page discovery, text extraction, pre-AI heuristics, AI classification batches, validation, page-level scoring, company-level aggregation, and Airtable/MongoDB storage.
  
  <img width="1670" height="294" alt="image" src="https://github.com/user-attachments/assets/945c0368-56ef-4144-8279-f25ad297b056" />

  ---

  ## Recent Reliability Update

  The workflow was recently updated to improve reliability, scoring quality, and reviewability across real brand tests.

  Current changes include:

  - Added page-level fetch validation after discovered-page HTTP requests so failed pages, 404s, and page-not-found responses do not reach AI classification or scoring.
  - Enabled non-fatal HTTP fetching for discovered pages, allowing one bad internal URL to be skipped without stopping the full assessment.
  - Added deterministic Normalize nodes after every AI batch to correct common LLM overclaims, including contact pages, event listings, generic blog indexes, medical case-study pages, and unsupported athlete/pickleball language.
  - Removed company-specific hardcoded cleanup language so normalization works across brands.
  - Preserved `ai_raw_output` alongside normalized fields for auditability.
  - Added batch metadata, text-length metadata, truncation flags, and page-level assessment keys for debugging and historical storage.
  - Added aggregate evidence counts such as `evidence_page_count`, `strong_evidence_page_count`, and `high_fit_page_count`.
  - Updated Airtable output so company-level assessments include partner group, evidence summaries, top evidence pages, source URLs, and scoring metadata.

  ---

  ## Storage

  ### MongoDB

  MongoDB is used for structured storage and historical evidence.

  Current brand assessment storage includes:

  - `brand_page_assessments`
  - `brand_company_assessments`
  - `brand_assessment_jobs`
  - `config`

  ### Airtable

  Airtable is used as the human-facing review layer.

  Current Airtable review output uses:

  - `brand_assessments`

  The `brand_assessments` table stores company-level sponsor-fit assessments.

  ---

  ## Example Input

  
  ### Brand Prospect Assessor Input
  
  ```json
  {
    "target_organization": "Brooklyn Pickleball Team",
    "target_org_type": "sports team",
    "target_market": "pickleball / sports / events",
    "analyzed_organization": "JOOLA",
    "analyzed_org_type": "brand",
    "root_domain": "https://joola.com",
    "partner_group": "brand_partner",
    "relationship_context": "Potential or comparable brand partner for Brooklyn Pickleball Team"
  }
  ```

  ---

  ## Example Output

  ### Brand Assessment Result

  ```json
  {
    "target_organization": "Brooklyn Pickleball Team",
    "analyzed_organization": "JOOLA",
    "partner_group": "brand_partner",
    "company_category": "pickleball equipment",
    "business_model": "consumer sports equipment",
    "recommended_offer": "player gear partnership",
    "brand_fit_score": 100,
    "brand_fit_conclusion": "strong brand fit",
    "recommended_action": "prioritize outreach",
    "primary_partnership_angle": "As a leading manufacturer of pickleball paddles and equipment, JOOLA is a natural partner for professional team sponsorship and gear supply.",
    "page_count": 10,
    "usable_page_count": 9,
    "evidence_page_count": 9,
    "aggregate_version": "brand_v1"
  }
  ```

  ---

  ## Review Output

  Company-level assessments are written to Airtable so results can be reviewed, sorted, filtered, and prioritized by score, partner group, category, and recommended action.

  <img width="1211" height="163" alt="airtable img" src="https://github.com/user-attachments/assets/0e4aef11-6081-4522-87d1-d93cdc99f516" />

  ---

  ## Current Test Results

  The workflow has been tested end-to-end on several partner categories using Brooklyn Pickleball Team as the target sports-property context.

  Note: These examples are prototype test runs using public-web data. They do not represent official partner recommendations or an affiliation with any listed organization.

  | Company | Category | Partner Group | Score | Conclusion |
  |---|---|---|---:|---|
  | JOOLA | pickleball equipment | brand_partner | 100 | strong brand fit |
  | KA-EX | wellness and recovery beverage | brand_partner | 56 | moderate brand fit |
  | SoftWave TRT | sports medicine and recovery | brand_partner | 54 | moderate brand fit |
  | Stella Blue Coffee | beverage / coffee | brand_partner | 39 | weak brand fit |

  These tests validate that direct pickleball equipment brands rank above adjacent recovery, beverage, and lifestyle brands for the Brooklyn Pickleball Team context.
  
  ---

  ## Key Capabilities

  - Multi-page brand/company assessment
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
  - Airtable output for review-ready assessments
  - MongoDB storage for page-level evidence and aggregate assessments
  - Early scan-memory and change-detection support

  ---

  ## Prototype History

  This project began as a sports-property signal scanner for Brooklyn Pickleball Team pages such as brand partners, past events, and latest news.

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

  - Add dedupe/upsert logic to the brand assessment workflow
  - Add material-change detection for repeated brand scans
  - Reduce AI token usage with tighter page selection and text limits
  - Batch assessment across multiple brands
  - Rank brands across sponsor categories
  - Add outreach-ready summaries
  - Add more Airtable review views for priority queues and follow-up tracking
  - Separate page-level and company-level MongoDB collections more cleanly
  - Test and harden job-status updates so completed queued jobs do not rerun automatically
  - Generalize Shopify utility-path exclusions such as `/collections/vendors`
  - Add local-market fit scoring for Brooklyn/NYC presence
  - Add partner-group-specific scoring for vendor, facility, experience, and brand partners
  - Add human-readable top-evidence summaries for Airtable

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

  The project combines public-web extraction, AI-assisted classification, deterministic scoring, scan memory, and structured storage to support practical partnership intelligence.

  The goal is to help identify which organizations and brands are actually worth reviewing, ranking, and prioritizing for partnership outreach.

