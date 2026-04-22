# Partnership Intelligence Workflow

An AI-powered workflow that converts unstructured partnership signals from public webpages into structured, decision-ready data for outreach prioritization and analysis.

> Designed as a backend data pipeline using n8n and LLMs.

> Built to solve the problem of extracting reliable partnership insights from noisy, unstructured public data using AI with validation and fallback logic.

> Designed to make AI outputs reliable and usable in real-world decision workflows.

---

## 📌 Overview

Sponsorship and partnership teams often rely on fragmented public information when identifying commercially relevant organizations. Public partnership signals such as sponsorship activity, brand visibility, and commercial expansion are often available online but remain unstructured and difficult to analyze consistently.

This workflow fetches source webpages, cleans and prepares content, classifies signals using AI, applies validation and fallback logic, and converts results into structured decision-ready records for analysis, scoring, and storage.

---

## ❗ What Problem It Solves

Teams working on partnerships and sponsorships often face:

- Unstructured data with limited actionable insights
- Unclear timing for outreach  
- Difficulty identifying partnership-ready organizations  
- Time-consuming manual research  
- Missed commercial signals in unstructured webpages and announcements  
- Limited clarity on decision-maker relevance  
- Inconsistent AI outputs when inputs are weak or incomplete  
- False positives from irrelevant or low-signal pages  

This workflow addresses these issues by classifying, validating, enriching, scoring, and storing signals in a consistent and usable format.

---

## ⚙️ Workflow Design

The workflow operates as a structured pipeline combining AI classification with validation and fallback handling:

1. Trigger workflow manually via n8n  
2. Input a source reference (`source_url`, `source_type`)  
3. Fetch webpage content from the provided source  
4. Clean and reduce raw HTML into AI-usable text and metadata  
5. Validate pre-AI input (`clean_text`) before extraction  
6. Run AI-based signal classification on cleaned source content  
7. Validate AI output for required classification fields  
8. Route incomplete or weak outputs through fallback handling  
9. Normalize the signal into a consistent schema  
10. Enrich the record with derived partnership fields  
11. Score the signal and generate a recommendation  
12. Store the final record in Airtable  
13. Store the final record in MongoDB as backup persistence  

---

## ⚙️ System Architecture

This workflow combines:

- Webpage ingestion from source URLs  
- Content cleanup and source preparation  
- AI-assisted classification using Groq / LLM  
- Conditional validation using IF nodes  
- Fallback handling for incomplete or irrelevant outputs  
- Schema normalization and enrichment  
- Signal scoring and recommendation generation  
- Airtable integration for primary structured storage  
- MongoDB integration for backup persistence  

---

## 🔄 Workflow Pipeline

![n8n workflow pipeline](https://github.com/user-attachments/assets/daf5b240-bb68-474e-837a-52cd524d5673)

## 📊 Airtable Output

![airtable output](https://github.com/user-attachments/assets/db06f03b-3561-4e0a-91a0-59e2c0da5209)

## 📊 MongoDB Output

![mongodb output](https://github.com/user-attachments/assets/be197168-7b19-4d36-815b-45471a2490eb)

### Key Capabilities

- Source ingestion from webpages  
- AI-based classification  
- Validation and fallback handling  
- Signal scoring and enrichment  
- Structured storage and persistence 

---

## 📥 Input Fields

- `source_url`
- `source_type`

---

The workflow derives downstream fields such as `organization`, `signal_text`, `date`, and `location` from fetched webpage content and AI extraction.

## 📤 Output Fields

- `organization`
- `org_type`
- `signal_text`
- `date`
- `location`
- `source_url`
- `source_type`
- `signal_type`
- `market_signal`
- `geography`
- `partnership_readiness`
- `sponsorship_readiness`
- `likely_partner_type`
- `priority`
- `blocker_type`
- `decision_maker_relevance`
- `signal_score`
- `signal_conclusion`
- `recommended_action`

---

## 🧪 Example Input

```json
{
  "source_url": "https://brooklynpickleballteam.com/brand-partners",
  "source_type": "company_announcement"
}
```

## 🧪 Example Output

```json
{
  "organization": "Brooklyn Pickleball Team",
  "org_type": "sports team",
  "signal_text": "Brooklyn Pickleball Team has a public partnership-related page indicating visible brand or team partnership activity.",
  "date": "2026-04-22",
  "location": "United States",
  "source_url": "https://brooklynpickleballteam.com/brand-partners",
  "source_type": "company_announcement",
  "signal_type": "partnership_presence",
  "market_signal": "neutral",
  "geography": "US",
  "partnership_readiness": "high",
  "sponsorship_readiness": "high",
  "likely_partner_type": "team partnership",
  "priority": "medium",
  "blocker_type": "unknown",
  "decision_maker_relevance": "commercial partnerships lead",
  "signal_score": 74,
  "signal_conclusion": "promising partnership signal",
  "recommended_action": "review and monitor"
}
```

## 📊 Current Status

This is an early-stage prototype focused on partnership and sponsorship intelligence workflows.

The current implementation includes:

- Source-driven webpage ingestion using public URLs  
- HTML cleanup and source preparation before AI extraction  
- AI-assisted signal classification using an LLM  
- Validation and fallback handling for incomplete, weak, or irrelevant outputs  
- Signal scoring and recommendation generation  
- Airtable for primary structured storage  
- MongoDB for backup persistence and local ownership of workflow output  

---

## 🔜 Next Steps

- Test against more real-world partnership, sponsorship, and irrelevant pages  
- Add organization tiering (local, regional, national, elite)  
- Expand scoring logic and company-level aggregation  
- Support multi-source checks for the same organization  
- Integrate external APIs or news sources for automated ingestion  
- Build a company-summary layer on top of individual signal checks  
- Add dashboards or reporting on top of Airtable and MongoDB data  

## 📝 Notes

This project demonstrates source-driven AI workflow design, structured signal enrichment, fallback-aware reliability, dual-storage persistence, and decision-ready partnership intelligence generation using n8n.


## 🚀 Why This Project Stands Out

- Combines AI with deterministic validation instead of relying solely on LLM output  
- Handles unreliable AI responses using fallback classification logic  
- Processes real-world unstructured data instead of static inputs  
- Produces decision-ready outputs instead of raw predictions  
- Designed as a reusable, scalable workflow pipeline  
