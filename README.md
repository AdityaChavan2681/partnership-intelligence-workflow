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

  Selected pages are sorted by AI priority, assigned into batches of two pages, and routed across five AI classification branches. Each branch classifies page-level sponsor-fit evidence using the same structured output schema.

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

  ---

  ## Architecture

  ```text
  Brand root domain
  → Homepage fetch
  → Internal link discovery
  → Relevant page filtering and role classification
  → Fetch selected pages, up to 10 total
  → Clean text extraction
  → Source metadata inference
  → Pre-AI heuristics and priority sorting
  → Batch assignment, 2 pages per AI branch
  → Five AI classification branches
  → Normalize and merge batch outputs
  → Validate page classifications
  → Fallback handling for weak or invalid AI output
  → Page-level sponsor-fit scoring
  → MongoDB page evidence storage
  → Brand-level aggregation
  → MongoDB aggregate assessment storage
  → Airtable review output
  ```

  ---

  ## Pipeline Snapshot

  The current n8n workflow includes page discovery, text extraction, pre-AI heuristics, AI classification batches, validation, page-level scoring, company-level aggregation, and Airtable/MongoDB storage.
  
  <img width="1503" height="598" alt="image 100" src="https://github.com/user-attachments/assets/b2737c16-f68c-453c-ac80-7b056944f5d4" />

  ---

  ## Storage

  ### MongoDB

  MongoDB is used for structured storage and historical evidence.

  Current brand assessment storage includes:

  - `brand_page_assessments`
  - aggregate brand assessment records

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
    "company_category": "pickleball equipment",
    "business_model": "consumer sports equipment",
    "recommended_offer": "product sponsorship",
    "brand_fit_score": 95,
    "brand_fit_conclusion": "strong brand fit",
    "recommended_action": "prioritize outreach",
    "primary_partnership_angle": "JOOLA is a strong equipment partner candidate based on direct pickleball product and facility partnership evidence.",
    "page_count": 4,
    "usable_page_count": 4,
    "aggregate_version": "brand_v1"
  }
  ```
  
  ---

  ## Key Capabilities

  - Multi-page brand/company assessment
  - Homepage-based internal page discovery
  - Source metadata inference
  - Clean text extraction from public webpages
  - AI classification with structured fields
  - Deterministic validation and fallback handling
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
  - Add stronger Airtable views for review queues
  - Separate page-level and company-level MongoDB collections more cleanly

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

