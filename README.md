# Partnership Intelligence Workflow

An n8n workflow that turns public partnership and sponsorship signals into structured intelligence for outreach prioritization, readiness analysis, and decision-maker relevance.

Built using n8n for workflow orchestration and rule-based signal enrichment.

## Overview

Sponsorship and partnership teams often deal with fragmented public information when trying to identify commercially relevant organizations. Signals like expansion, partnership activity, sponsorship intent, and commercial growth are publicly visible, but they are noisy, inconsistent, and difficult to compare quickly.

This workflow converts those public signals into structured records that can support faster research, better prioritization, and more context-aware outreach.

## What Problem It Solves

Many sponsorship and partnership teams face the same recurring issues:

- too much public information, but not enough structured insight
- unclear timing for outreach
- difficulty identifying which organizations are actually partnership-ready
- manual research takes too long
- commercial blockers are easy to miss in raw announcements
- decision-maker relevance is often unclear from surface-level information

This workflow helps reduce that gap by classifying and enriching public signals into a format that is easier to compare and act on.

## Current Workflow

1. Trigger workflow manually using n8n
2. Input structured sponsorship signal data
3. Classify signal type, market signal, and geography
4. Enrich partnership and sponsorship readiness based on defined rules

## ⚙️ n8n Workflow Implementation

This workflow is implemented using n8n for orchestration and rule-based enrichment.

### 🔄 Workflow Pipeline

<img width="1152" height="284" alt="image" src="https://github.com/user-attachments/assets/0edcef18-a28f-4049-b282-a132dcc5cd07" />

The workflow processes structured input signals through classification and enrichment stages to generate decision-ready outputs.

This pipeline demonstrates:
- Manual trigger with structured input schema
- Signal classification using rule-based logic
- Partnership readiness enrichment
- Transformation into structured output fields for analysis

## Input Fields

- `organization`
- `org_type`
- `signal_text`
- `date`
- `location`
- `source_url`
- `source_type`

## Output Fields

- `signal_type`
- `market_signal`
- `geography`
- `partnership_readiness`
- `sponsorship_readiness`
- `likely_partner_type`
- `priority`
- `blocker_type`
- `decision_maker_relevance`

## Example Input

```json
{
  "organization": "Brooklyn Pickleball Team",
  "org_type": "sports team",
  "signal_text": "Brooklyn Pickleball announced expanded partnership outreach and commercial growth efforts ahead of the new season.",
  "date": "2026-04-20",
  "location": "United States",
  "source_url": "https://brooklynpickleballteam.com/brand-partners",
  "source_type": "company_announcement"
}
```

## Example Output

```json
{
  "organization": "Brooklyn Pickleball Team",
  "org_type": "sports team",
  "signal_text": "Brooklyn Pickleball announced expanded partnership outreach and commercial growth efforts ahead of the new season.",
  "date": "2026-04-20",
  "location": "United States",
  "source_url": "https://brooklynpickleballteam.com/brand-partners",
  "source_type": "company_announcement",
  "signal_type": "expansion",
  "market_signal": "growth",
  "geography": "US",
  "partnership_readiness": "high",
  "sponsorship_readiness": "high",
  "likely_partner_type": "season sponsorship",
  "priority": "high",
  "blocker_type": "unknown",
  "decision_maker_relevance": "commercial partnerships lead"
}
```

## 📊 Current Status

This is an early-stage prototype focused on sports sponsorship and partnership intelligence.  
The current version uses manually provided public signals and rule-based enrichment inside n8n to generate structured outputs.

## 🔜 Next Steps

- Test more real-world sports and partnership signals
- Add organization tiering (local, regional, national, elite)
- Expand blocker detection and commercial maturity logic
- Integrate external sources through HTTP/API nodes
- Store outputs in Airtable, Google Sheets, or a database

## 📝 Notes

This project was built as a proof-of-work prototype for AI workflow design, structured signal enrichment, and partnership intelligence use cases.
