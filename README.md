  # Partnership Intelligence Pipeline

  ### Sports Property Signal Scanning, Brand Prospect Assessment, and Scan Memory

  A public-web intelligence pipeline for turning fragmented partnership and sponsorship evidence into structured opportunity intelligence.

  The project contains two related n8n workflows:

  1. **Prototype Sports Property Signal Scanner**
     Scans public pages from a sports organization to detect partnership readiness, sponsorship signals, commercial activity, and outreach priority.

  2. **Brand Prospect Assessor**
     Scans public pages from an outside brand/company website, evaluates page-level sponsor-fit evidence, and aggregates the result into a company-level brand assessment.

  Together, these workflows support both sides of partnership intelligence:

  - Is a sports property commercially active and partnership-ready?
  - Is a brand a strong sponsor or partner prospect for that property?

  ---

  ## Overview

  Partnership teams often rely on scattered public signals: partner pages, event pages, news posts, product pages, facility pages, and brand websites. These signals are noisy, inconsistent, and difficult to compare manually.

  This project converts that public evidence into structured records that can be scored, reviewed, stored, and monitored over time.

  The pipeline uses a hybrid approach:

  - deterministic source heuristics
  - AI-assisted classification
  - validation and fallback handling
  - page-level scoring
  - company-level aggregation
  - MongoDB storage
  - Airtable surfacing for review-ready outputs

  ---

  ## Workflows

  ### Prototype Sports Property Signal Scanner

  The original workflow scans a sports organization’s own public pages.

  Example sources:

  - partner pages
  - past event pages
  - latest news pages
  - sponsorship or commercial pages

  It evaluates whether the organization shows signs of commercial readiness, partnership activity, sponsorship potential, or outreach priority.

  ### Main Outputs

  - `signal_type`
  - `market_signal`
  - `partnership_readiness`
  - `sponsorship_readiness`
  - `likely_partner_type`
  - `priority`
  - `signal_score`
  - `signal_conclusion`
  - `recommended_action`
  - `scan_status`
  - `storage_reason`

  This workflow also includes early scan-memory logic for comparing current results against previous scans.

  ---

  ### Brand Prospect Assessor

  The newer workflow evaluates an outside brand or company as a potential sponsor or partner.

  Starting from a brand root domain, it discovers and evaluates relevant pages such as:

  - homepage
  - about page
  - product/category pages
  - partner/sponsorship pages
  - blog/news pages
  - events/athlete pages
  - contact pages as supporting context only

  Each page is classified and scored individually. The workflow then aggregates the strongest page-level evidence into one company-level sponsor-fit assessment.

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
  Input
  → Page discovery / source selection
  → Page fetch
  → Text extraction
  → Source metadata inference
  → Pre-AI heuristics
  → AI classification
  → Validation and fallback handling
  → Page-level scoring
  → Key/signature generation
  → MongoDB storage
  → Brand-level aggregation
  → Airtable review output
  ```

  ## Old Pipeline Snapshot

  <img width="1195" height="259" alt="image 102" src="https://github.com/user-attachments/assets/bc0b5342-5501-4d0a-b064-63ce0f7d2ef7" />

  ## Updated Pipeline Snapshot

  The current n8n workflow includes page discovery, text extraction, pre-AI heuristics, AI classification batches, validation, page-level scoring, company-level aggregation, and Airtable/MongoDB storage.

  <img width="1503" height="598" alt="image 100" src="https://github.com/user-attachments/assets/b2737c16-f68c-453c-ac80-7b056944f5d4" />


  ---

  ## Storage

  ### MongoDB

  MongoDB is used for structured storage and historical evidence.

  Current collections include:
  
  - `signal_checks`
  - `scan_history`
  - `brand_page_assessments`
  - aggregate brand assessment records

  ### Brand Page Assessment Result

  MongoDB stores both detailed page-level records and broader assessment outputs.
  ```json
  {
    "target_organization": "Brooklyn Pickleball Team",
    "analyzed_organization": "JOOLA",
    "source_url": "https://joola.com/",
    "page_role": "company_profile",
    "company_category": "pickleball equipment",
    "business_model": "consumer sports equipment",
    "sports_relevance": "high",
    "pickleball_relevance": "high",
    "sponsor_fit": "high",
    "partnership_angle": "Equipment and apparel partnership for pickleball events and players",
    "outreach_priority": "high",
    "recommended_offer": "product sponsorship",
    "brand_fit_score": 100,
    "brand_fit_conclusion": "strong brand fit",
    "recommended_action": "prioritize outreach",
    "storage_tier": "priority_brand_prospect"
  }
  ```
  ### Airtable

  Airtable is used as the human-facing review layer.

  Current Airtable tables include:

  - `signals`
  - `brand_assessments`

  The `signals` table stores sports-property signal results.
  The `brand_assessments` table stores company-level sponsor-fit assessments.

  <img width="1925" height="886" alt="image 101" src="https://github.com/user-attachments/assets/33279c7e-f74b-4e83-bd59-3d9bec36d9b4" />

  ---

  ## Example Inputs

  
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

  ### Prototype Sports Property Signal Input


  ```json
  {
    "organization": "Brooklyn Pickleball Team",
    "org_type": "sports team",
    "sources": [
      {
        "source_url": "https://brooklynpickleballteam.com/brand-partners"
      },
      {
        "source_url": "https://brooklynpickleballteam.com/past-events"
      },
      {
        "source_url": "https://brooklynpickleballteam.com/latest-news"
      }
    ]
  }
  ```


  ---

  ## Example Outputs

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

  ### Prototype Sports Property Signal Result

  ```json
  {
    "organization": "Brooklyn Pickleball Team",
    "source_type": "brand_partners_page",
    "signal_type": "partnership_presence",
    "partnership_readiness": "high",
    "sponsorship_readiness": "high",
    "likely_partner_type": "brand partnership",
    "signal_score": 96,
    "signal_conclusion": "strong partnership signal",
    "recommended_action": "prioritize outreach"
  }
  ```
  
  ---

  ## Key Capabilities

  - Multi-source sports property signal scanning
  - Multi-page brand/company assessment
  - Homepage-based brand page discovery
  - Source metadata inference
  - Clean text extraction from public webpages
  - AI classification with structured fields
  - Deterministic validation and fallback handling
  - Page-level opportunity scoring
  - Company-level sponsor-fit aggregation
  - Airtable output for review-ready records
  - MongoDB storage for evidence and history
  - Early scan-memory and change-detection support

  ---

  ## Current Status

  This is an active prototype.

  The original sports-property signal scanner was intentionally hard-coded around a few Brooklyn Pickleball Team pages. That prototype helped establish the core system patterns:

  - source metadata
  - clean text extraction
  - AI classification
  - fallback handling
  - scan-memory signatures

  The newer brand prospect assessor generalizes the system toward reusable sponsor-prospect evaluation across outside companies and brands.

  ---

  ## Recent Progress

  This update marks a major expansion of the project into reusable brand prospect intelligence.

  Today’s work focused on building and refining the **Brand Prospect Assessor**, a workflow that evaluates outside brands and companies as potential sponsors or partners for a target sports property.

  Instead of relying on a fixed list of known source pages, the new workflow starts from a company root domain, discovers relevant internal pages, evaluates each page as evidence, and aggregates the results into one company-level
  sponsor-fit assessment.

  Key progress from this update:

  - Built a dedicated brand/company assessment workflow
  - Added homepage-based internal page discovery
  - Added page role classification for homepage, about, product, blog/news, sponsorship, athlete/event, and contact pages
  - Added page-level sponsor-fit classification
  - Added page-level brand-fit scoring
  - Added fallback handling when AI output is missing or invalid
  - Added company-level aggregation across multiple page assessments
  - Added MongoDB storage for page-level evidence
  - Added MongoDB storage for aggregate brand assessment records
  - Added a dedicated Airtable `brand_assessments` table for review-ready company outputs
  - Updated Airtable field names to match brand assessment outputs
  - Renamed workflow nodes for clearer page-level and company-level stages
  - Identified Groq token usage as the next optimization target

  The main shift is that the system can now answer a broader and more valuable question:

  ```text
  Given a target sports property and an outside company,
  is this company a credible sponsor or partner prospect?
  ```
  This turns the project from a simple signal scanner into a reusable sponsor-prospect assessment pipeline. It now supports structured evaluation of brands across multiple public pages, with page-level evidence, final company-level
  scoring, and storage paths for both detailed review and high-level decision making.
  
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

  - Partnership target discovery
  - Sponsor prospect qualification
  - Sports property commercial-readiness scanning
  - Brand fit scoring
  - Outreach prioritization
  - Sponsorship category research
  - Historical monitoring of public commercial signals

  ## Why This Matters

  This is more than a scraping workflow.

  The project combines public-web extraction, AI-assisted classification, deterministic scoring, scan memory, and structured storage to support practical partnership intelligence.

  The goal is to help identify which organizations and brands are actually worth reviewing, ranking, and prioritizing for partnership outreach.

