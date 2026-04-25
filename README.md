# Partnership Intelligence Pipeline
### Sports Property Signal Scanning, Brand Fit Assessment, and Scan Memory

An intelligence pipeline that transforms fragmented public partnership and sponsor signals into structured opportunity intelligence.

The project contains two connected workflows:

1. Sports Property Signal Scanner: evaluates multiple public pages from a sports organization to detect partnership readiness.
2. Brand Prospect Assessor: evaluates multiple pages from a brand/company website, scores page-level evidence, and aggregates the results into a company-level sponsor-fit assessment.


---

## 📌 Overview

The system evaluates public web evidence from two directions.

The sports property workflow asks whether an organization shows commercial partnership readiness across pages such as partner lists, events, and news.

The brand assessment workflow asks whether a company is a strong sponsor or partner prospect by analyzing its homepage, about page, product/category page, and blog/news content.

Together, the workflows support both opportunity detection and brand-prospect qualification.


---

### Key Capabilities
- Sports property signal scanning across partner, event, and news pages
- Brand prospect assessment across homepage, about, product, and blog pages
- AI extraction with deterministic validation and fallback handling
- Page-level scoring plus company-level sponsor-fit aggregation
- MongoDB scan memory for historical comparison and structured storage 

---

## ❗ Problem

Partnership teams often rely on fragmented public signals that are noisy,
difficult to compare, and often too weak to support reliable outreach decisions.

This system converts scattered public evidence into structured opportunity
intelligence that can be evaluated, ranked, and prioritized for outreach.

- Multi-source signal comparison  
- AI-assisted extraction with deterministic validation
- Opportunity scoring  
- Scan-memory and change detection  
- Tiered storage for surfaced vs historical intelligence

---

## ⚙️ Pipeline Stages

The system operates through two related pipelines.

### Sports Signal Scanner
1. Source registry creation
2. Page fetching and text extraction
3. Source qualification and AI classification
4. Signal normalization and scoring
5. Scan-memory comparison
6. Tiered Airtable/MongoDB storage

### Brand Prospect Assessor
1. Brand input creation
2. Brand page candidate generation
3. Page fetching and text extraction
4. Page-level brand fit classification
5. Page-level scoring and MongoDB storage
6. Company-level aggregation and final sponsor-fit assessment


---

## 🧠 Architecture Layers

### Source Preparation
- Multi-source ingestion
- AI classification with deterministic validation
- Source-level scoring and recommendations

### Evaluation
- Source heuristics and company evaluation framework
- Scan-memory and deduplication
- Partner reference validation

### Intelligence & Storage
- MongoDB for page-level evidence, scan history, and company-level assessments
- Airtable for surfaced high-priority opportunity signals
- `brand_page_assessments` for individual brand page evidence
- `brand_company_assessments` for aggregated company-level sponsor-fit decisions

---

## 🔄 Workflow Pipeline

![n8n workflow pipeline](https://github.com/user-attachments/assets/7972bfcb-caf5-46c9-afd9-92aa220bded7)

## 📊 Airtable Output

Strong Source Result

![airtable output](https://github.com/user-attachments/assets/b224d2fc-6283-4a28-982f-a11dab598f7d)

## 📊 MongoDB Output

Moderate Source Result

![mongodb output1](https://github.com/user-attachments/assets/6870d254-31b5-4d47-9e28-d6a04b3f35a1)

Weak Source Result

![mongodb output2](https://github.com/user-attachments/assets/34ddf0a8-d5c6-4445-a356-9642e7efce08)

---

## ✅ Current Capabilities

- Multi-source signal ingestion for sports properties
- Multi-page brand assessment for sponsor prospects
- AI-assisted classification with deterministic validation
- Page-level and company-level scoring
- MongoDB storage for scan history, page evidence, and final company assessments
- Optional Airtable routing for surfaced high-priority signals

---

## 🔬 Advanced Workflow Node Detail (Optional Deep Dive)

### Source Preparation
- Manual Test Trigger
- Create Source Registry
- Select Candidate Sources
- Infer Source Metadata
- Fetch Candidate Pages
- Extract Clean Source Text
- Check AI Readiness

### Evaluation
- Apply Source Heuristics
- Apply Company Evaluation Framework
- Classify Partnership Signal
- Validate AI Output
- Assemble Signal Record
- Normalize And Enrich Signal
- Score Opportunity Signal

### Intelligence & Storage
- Compute Scan Keys
- Find Existing Scan Record
- Merge Current Scan With History
- Classify Scan Status
- Route Airtable Or Mongo
- Save To Airtable
- Save Backup To MongoDB

---


## 📥 Example Inputs

The project supports two input shapes depending on the workflow.

### Sports Property Signal Scanner

This workflow starts from an organization-level source registry containing a `sources[]` array of candidate URLs.

### Brand Prospect Assessor

This workflow starts from one brand/company root domain, generates candidate pages, scores each page, and aggregates the results into one company-level assessment.

These source records form the signal comparison set used for evaluation and scoring.

---

### Sports Property Signal Scanner

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

### Brand Prospect Assessor

```json
{
  "target_organization": "Brooklyn Pickleball Team",
  "target_org_type": "sports team",
  "target_market": "pickleball / sports / events",
  "analyzed_organization": "JOOLA",
  "analyzed_org_type": "brand",
  "root_domain": "https://joola.com",
  "relationship_context": "Existing or comparable brand partner for Brooklyn Pickleball Team"
}
```

📤 Output Fields

Sports Property Signal Fields
signal_type
signal_score
signal_conclusion
recommended_action
partnership_readiness
sponsorship_readiness
scan_key
scan_status
storage_reason
Brand Page Assessment Fields
page_label
source_role
brand_fit_score
brand_fit_conclusion
company_category
sponsor_fit
partnership_angle
recommended_offer
Company-Level Brand Assessment Fields
overall_company_category
overall_sponsor_fit
company_fit_score
best_partnership_angle
final_recommendation
company_assessment_key
storage_tier

---

## 📤 Output Fields

### Sports Property Signal Fields
- `signal_type`
- `signal_score`
- `signal_conclusion`
- `recommended_action`
- `partnership_readiness`
- `sponsorship_readiness`
- `scan_key`
- `scan_status`
- `storage_reason`

### Brand Page Assessment Fields
- `page_label`
- `source_role`
- `brand_fit_score`
- `brand_fit_conclusion`
- `company_category`
- `sponsor_fit`
- `partnership_angle`
- `recommended_offer`

### Company-Level Brand Assessment Fields
- `overall_company_category`
- `overall_sponsor_fit`
- `company_fit_score`
- `best_partnership_angle`
- `final_recommendation`
- `company_assessment_key`
- `storage_tier`

---

## 📤 Example Outputs

### Sports Property Signal Result

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

### Company-Level Brand Assessment

```json
{
  "target_organization": "Brooklyn Pickleball Team",
  "analyzed_organization": "JOOLA",
  "page_count": 4,
  "average_page_score": 95,
  "overall_company_category": "pickleball equipment",
  "overall_sponsor_fit": "high",
  "company_fit_score": 95,
  "best_partnership_angle": "Official pickleball paddle and equipment partner",
  "recommended_offer": "product sponsorship",
  "final_recommendation": "prioritize outreach",
  "storage_tier": "priority_company_prospect"
}
```

## 🧪 Example Use Case

This helps answer:

- Which public source actually shows partnership readiness?
- Which pages are useful evidence versus noise?
- Is this organization worth prioritizing for outreach?
- Which source should a human reviewer trust most?

## 📊 Current Status

Current prototype supports:

- Multi-source signal ingestion for sports properties
- AI-assisted classification with validation and fallback handling
- Source-level scoring and scan-memory signatures
- Multi-page brand prospect assessment
- Page-level MongoDB storage in `brand_page_assessments`
- Company-level aggregation in `brand_company_assessments`
- Optional Airtable routing for surfaced high-priority signals

## 🔜 Roadmap

- Dedupe and upsert logic for page-level and company-level assessments
- Dynamic brand page discovery from homepage links
- Batch assessment across multiple brands
- Company-level ranking across sponsor categories
- Historical material-change detection
- Outreach-ready recommendation generation

## 🧠 Design Principles

- Deterministic validation over blind AI trust
- Multi-source evidence over single-source assumptions
- Structured intelligence over raw extraction

## 🚀 Why This Is More Than a Workflow

This prototype combines:

- Signal extraction  
- Decision scoring  
- Scan memory  
- Historical intelligence storage  

It is designed to support opportunity discovery,
prioritization, and evolving partnership intelligence —
not just data processing.

## 🎯 Potential Real-World Applications

- Partnership target discovery
- Sponsor opportunity ranking
- Multi-source outreach intelligence
- Signal monitoring across target organizations
