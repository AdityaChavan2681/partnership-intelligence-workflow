# Multi-Source Partnership Intelligence Workflow

An AI-powered workflow that transforms unstructured partnership signals from public webpages into structured, decision-ready data using multi-source comparison.

> Built to identify commercially relevant organizations from fragmented public data without relying on a single weak source.

> Designed to make AI outputs reliable using validation, fallback logic, and multi-source signal comparison.

---

## 📌 Overview

Partnership and sponsorship teams often rely on scattered public information when deciding who is worth outreach. Useful signals such as public partner listings, event activity, and news visibility may exist online, but they are often noisy, incomplete, and hard to compare consistently.

This workflow improves that process by evaluating multiple public sources for a single organization, comparing signal strength, and generating structured, decision-ready outputs.

Instead of trusting one page alone, the workflow compares a **strong**, **medium**, and **weak** source for the same organization and produces one structured record per source.

---

## ❗ What Problem It Solves

Teams working on partnerships and sponsorships often face:

- Unstructured public data with limited actionable insight
- Over-reliance on single webpages that may be weak or misleading
- Time-consuming manual research across multiple public pages
- Difficulty distinguishing strong commercial evidence from noise
- Missed outreach opportunities because signals are not compared consistently
- False positives from irrelevant or low-signal pages
- Inconsistent AI outputs when source content is weak or incomplete
- Limited clarity on whether an organization is truly partnership-ready

This workflow addresses those issues by turning multiple public sources into structured, comparable records with validation, fallback logic, and score-based recommendations.

---

## ⚙️ Workflow Design

The workflow operates as a source-aware pipeline:

1. Trigger the workflow manually in n8n
2. Seed candidate public sources for one organization
3. Select a small source set for comparison
4. Fetch webpage content for each selected source
5. Clean raw HTML into compact source text and metadata
6. Validate whether each source is ready for AI extraction
7. Classify partnership signals using AI
8. Validate AI output fields
9. Route failed or incomplete results through fallback handling
10. Merge AI output back with source metadata
11. Normalize and enrich the signal record
12. Score each source independently
13. Store final records in Airtable
14. Store backup records in MongoDB

---

## ⚙️ System Architecture

This workflow combines:

- Candidate source seeding for one organization
- Multi-source webpage ingestion
- Content cleanup and source preparation
- AI-assisted partnership signal classification
- IF-node validation before and after AI extraction
- Fallback handling for incomplete or invalid outputs
- Record normalization and enrichment
- Source-level scoring and recommendation generation
- Airtable integration for primary structured storage
- MongoDB integration for backup persistence

---

## 🔄 Workflow Pipeline

![n8n workflow pipeline](https://github.com/user-attachments/assets/1ce3368d-a19d-4583-afa9-f8a7609e3507)

## 📊 Airtable Output

![airtable output](https://github.com/user-attachments/assets/e0247fbd-3e07-457c-afe7-6a620468b5cc)


## 📊 MongoDB Output

![mongodb output](https://github.com/user-attachments/assets/51c7b316-a3e7-4381-ba2d-0a8e2a76630a)

## ✅ Current Capabilities

- Seed multiple public sources for one organization
- Compare different source types for the same company
- Fetch and clean public webpage content
- Preserve source metadata across the workflow
- Classify partnership signals using AI
- Validate AI readiness and AI output
- Handle incomplete results through fallback routing
- Normalize and enrich structured signal records
- Score strong, medium, and weak sources separately
- Store multiple source records per organization in Airtable
- Store backup records in MongoDB

---

### 🧩 Workflow Nodes

- `Manual Test Trigger`
- `Create Source Registry`
- `Select Candidate Sources`
- `Fetch Candidate Pages`
- `Extract Clean Source Text`
- `Check AI Readiness`
- `Classify Partnership Signal`
- `Validate AI Output`
- `Assemble Signal Record`
- `Assemble Fallback Record`
- `Normalize And Enrich Signal`
- `Score Opportunity Signal`
- `Save To Airtable`
- `Save Backup To MongoDB`

## 📥 Input Structure

The workflow no longer starts from a single source URL alone.

It now uses a structured source set like:

- `organization`
- `org_type`
- `source_url`
- `source_type`
- `source_platform`
- `source_trust_level`
- `signal_strength_bucket`

These source fields help the workflow compare different public sources before final scoring.

---

## 📥 Example Input

```json
[
  {
    "organization": "Brooklyn Pickleball Team",
    "org_type": "sports team",
    "source_url": "https://brooklynpickleballteam.com/brand-partners",
    "source_type": "brand_partners_page",
    "source_platform": "website",
    "source_trust_level": "high",
    "signal_strength_bucket": "strong"
  },
  {
    "organization": "Brooklyn Pickleball Team",
    "org_type": "sports team",
    "source_url": "https://brooklynpickleballteam.com/past-events",
    "source_type": "event_history_page",
    "source_platform": "website",
    "source_trust_level": "medium",
    "signal_strength_bucket": "medium"
  },
  {
    "organization": "Brooklyn Pickleball Team",
    "org_type": "sports team",
    "source_url": "https://brooklynpickleballteam.com/latest-news",
    "source_type": "news_hub_page",
    "source_platform": "website",
    "source_trust_level": "medium",
    "signal_strength_bucket": "weak"
  }
]
```

📤 Output Fields

Each processed source produces a structured record with fields such as:

- organization  
- org_type  
- signal_text  
- date  
- location  
- source_url  
- source_type  
- source_platform  
- source_trust_level  
- signal_strength_bucket  
- signal_type  
- market_signal  
- geography  
- partnership_readiness  
- sponsorship_readiness  
- likely_partner_type  
- priority  
- blocker_type  
- decision_maker_relevance  
- signal_score  
- signal_conclusion  
- recommended_action  

---

📤 Example Output

Strong source result

```json
{
  "organization": "Brooklyn Pickleball Team",
  "org_type": "sports team",
  "signal_text": "Brooklyn Pickleball Team - Brand Partners - Join the Brooklyn Pickleball Team - Brand Partners Register for the Brooklyn Pickleball Team Minor League DUPR 12-20 Rosters More Our Partners We are proud to partner with the following organizations !",
  "date": "2026-04-22",
  "location": "United States",
  "source_url": "https://brooklynpickleballteam.com/brand-partners",
  "source_type": "brand_partners_page",
  "source_platform": "website",
  "source_trust_level": "high",
  "signal_strength_bucket": "strong",
  "signal_type": "partnership_presence",
  "market_signal": "neutral",
  "geography": "US",
  "partnership_readiness": "high",
  "sponsorship_readiness": "high",
  "likely_partner_type": "brand partnership",
  "priority": "medium",
  "blocker_type": "unknown",
  "decision_maker_relevance": "commercial partnerships lead",
  "signal_score": 92,
  "signal_conclusion": "strong partnership signal",
  "recommended_action": "prioritize outreach"
}
```

Medium source result

```json
{
  "organization": "Brooklyn Pickleball Team",
  "org_type": "sports team",
  "source_url": "https://brooklynpickleballteam.com/past-events",
  "source_type": "event_history_page",
  "signal_type": "partnership_presence",
  "market_signal": "growth",
  "partnership_readiness": "high",
  "sponsorship_readiness": "high",
  "likely_partner_type": "team partnership",
  "signal_score": 89,
  "signal_conclusion": "strong partnership signal",
  "recommended_action": "prioritize outreach"
}
```

Weak source result

```json
{
  "organization": "Brooklyn Pickleball Team",
  "org_type": "sports team",
  "source_url": "https://brooklynpickleballteam.com/latest-news",
  "source_type": "news_hub_page",
  "signal_type": "other",
  "market_signal": "neutral",
  "partnership_readiness": "low",
  "sponsorship_readiness": "low",
  "likely_partner_type": "unknown",
  "signal_score": 2,
  "signal_conclusion": "weak signal",
  "recommended_action": "monitor only"
}
```

🧪 Example Use Case

For one organization, the workflow compares:

- A strong commercial source (partners page)
- A medium-signal source (event history page)
- A weak or noisy source (news hub page)

This helps answer:

Which public source actually shows partnership readiness?
Which pages are useful evidence vs noise?
Is this organization worth prioritizing for outreach?
Which source should a human reviewer trust most?


📊 Current Status

This project is an early but working prototype of a source-aware partnership intelligence pipeline.

The current implementation includes:

- Multi-source public webpage ingestion for one organization  
- Source metadata preservation across the workflow  
- HTML cleanup and AI-ready text extraction  
- AI-assisted classification using Groq / LLM  
- Input and output validation around AI extraction  
- Fallback handling for invalid or incomplete responses  
- Record normalization and enrichment  
- Comparative source-level scoring  
- Airtable as primary structured storage  
- MongoDB as backup persistence  


🔜 Next Steps

Expand the source registry beyond strong / medium / weak starter sources
Add organization-level rollups across all source records
Add source-selection rules from a database instead of hardcoded inputs
Add sponsor-tier and media-presence comparison logic
Add event-richness variables such as event count, guest quality, and media coverage
Support batch processing across multiple organizations
Add dashboards or reporting on top of Airtable and MongoDB
Move stable business rules from code nodes into config tables or services


📝 Notes

This project demonstrates:

- Workflow orchestration using n8n  
- AI extraction with deterministic validation  
- Multi-source signal comparison  
- Fallback-safe record generation  
- Source-level scoring and recommendation logic  
- Structured storage for downstream analysis  

It started as a single-source extraction workflow and evolved into a multi-source comparison pipeline for partnership intelligence.


🚀 Why This Project Stands Out

- Moves beyond single-page AI extraction into multi-source comparison  
- Combines AI with deterministic validation instead of trusting LLM output blindly  
- Handles weak and invalid cases through fallback logic  
- Preserves source metadata for explainability and scoring  
- Produces decision-ready records instead of raw AI responses  
- Demonstrates real workflow design, not just prompt experimentation  
- Can evolve into a larger source registry and company-level intelligence system  
