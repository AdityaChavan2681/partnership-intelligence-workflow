# Multi-Source Partnership Intelligence Pipeline
### Signal Scoring, Scan Memory, and Tiered Storage

An intelligence pipeline that transforms fragmented public partnership signals
into structured opportunity intelligence through multi-source comparison,
deterministic validation, signal scoring, and scan-memory.

> Built to surface commercially relevant organizations from noisy public signals.

> Designed to combine AI extraction, deterministic validation, and historical scan memory for more reliable decision support.

---

## 📌 Overview

Partnership decisions often rely on fragmented public signals that are noisy and difficult to compare.

This system evaluates multiple public sources for the same organization and transforms them into ranked, decision-ready partnership intelligence.

Rather than trusting a single page, the pipeline compares strong, medium, and weak signals to support more reliable partnership targeting.

---

### Key Capabilities
- Multi-source signal evaluation
- AI + deterministic validation
- Scan-memory and change detection
- Opportunity scoring and prioritization

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

The system operates through six stages:

1. Source Discovery  
2. Signal Extraction and Source Qualification
3. AI + Rule-Based Evaluation  
4. Validation and Fallback Handling  
5. Scoring, Scan Memory, and Change Detection  
6. Opportunity Surfacing and Tiered Storage

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
- Airtable for surfaced opportunity signals
- MongoDB for historical scan intelligence

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

- Multi-source signal comparison
- Validation and fallback-safe processing
- Source-level scoring
- Scan-memory and deduplication
- Tiered Airtable / Mongo storage
- Opportunity scoring and recommendation generation
- Partner reference dataset integration (prototype)

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

## 📥 Input Structure

The workflow no longer starts from a single source URL alone.

It now starts from an organization-level source registry containing a `sources[]` array of candidate URLs. Source metadata is inferred later in the pipeline.

Initial input fields:

- `organization`
- `org_type`
- `sources[]`

Derived later in the workflow:

- `source_type`
- `source_platform`
- `source_trust_level`
- `signal_strength_bucket`


These source records form the signal comparison set used for evaluation and scoring.

---

## 📥 Example Input

```json
[
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
]
```

## 📤 Output Fields

### Core Signal Fields
- `signal_type`
- `signal_score`
- `signal_conclusion`
- `recommended_action`
- `partnership_readiness`
- `sponsorship_readiness`

### Intelligence Fields
- `scan_key`
- `result_signature`
- `storage_tier`
- `scan_status`
- `storage_reason`

### Evaluation Fields
- `page_role`
- `commercial_relevance`
- `partnership_visibility`
- `outreach_readiness_hint`

---

## 📤 Example Output 

Strong source result (Airtable)

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

Moderate source result (MongoDB)

```json
{
  "organization": "Brooklyn Pickleball Team",
  "source_type": "event_history_page",
  "signal_type": "partnership_push",
  "signal_score": 48,
  "signal_conclusion": "moderate signal",
  "recommended_action": "monitor for stronger evidence",
  "storage_tier": "mongo_only",
  "scan_status": "new",
  "storage_reason": "new_but_low_priority"
}
```

Weak Source Result (MongoDB)

```json
{
  "organization": "Brooklyn Pickleball Team",
  "source_type": "news_hub_page",
  "signal_type": "other",
  "signal_score": 0,
  "signal_conclusion": "weak signal",
  "recommended_action": "monitor only",
  "storage_tier": "mongo_only",
  "scan_status": "new",
  "storage_reason": "new_but_low_priority"
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

- Multi-source signal ingestion
- AI-assisted classification with validation
- Source-level scoring
- Scan-memory support through MongoDB lookup and stored scan signatures
- Tiered storage with Airtable for surfaced signals and MongoDB for historical/reference records

## 🔜 Roadmap

- Research-backed partner reference scoring
- Multi-organization opportunity ranking
- Historical material-change detection
- Batch intelligence across target portfolios


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
