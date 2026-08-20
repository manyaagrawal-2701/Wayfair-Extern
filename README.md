# Wayfair Rugs Market Intelligence — AI Agent System

An n8n-based pipeline of connected AI agents that automates market trend discovery, competitor benchmarking, and marketing content generation for Wayfair's Rugs category team — consolidated into a single, actionable dashboard.

**Author:** Manya Agrawal
**Cohort:** June 8 (Wayfair Externship)

---

## 1. Problem

Wayfair's Rugs category team spends significant manual effort on three recurring tasks:
- Researching what's trending in the market (colors, materials, styles)
- Benchmarking Wayfair's pricing and assortment against competitors (Amazon, Walmart)
- Writing on-brand marketing content based on those insights

This is slow, repetitive, and has to be redone for every product category. This project automates the entire pipeline — from raw market data to a ready-to-use dashboard and marketing content.

---

## 2. System Overview

The system is composed of **4 connected AI agents** plus **1 dashboard builder**, each implemented as a standalone n8n workflow:

| # | Workflow | Input | Output |
|---|----------|-------|--------|
| 1 | **Moodboard Generator** | Short design prompt (e.g. "bohemian rugs, neutral tones") | AI-generated visual moodboard |
| 2 | **Trend Discovery Agent** | Product category (e.g. "Outdoor Rugs") | HTML market trend report |
| 3 | **Competitor Monitoring Agent** | Product category | HTML competitor intelligence report |
| 4 | **AI Insights & Content Agent** | P2 + P3 reports, category, focus area, audience, season | HTML marketing content strategy |
| 5 | **Dashboard Builder** | P2 + P3 reports, category | Unified HTML Category Intelligence Dashboard |

**Data flow:** Agent 2 and Agent 3 run independently → their outputs feed into both Agent 4 (content generation) and the Dashboard Builder (data consolidation).

---

## 3. Agent Details

### Agent 1 — Moodboard Generator
- **Objective:** Transform a short text prompt into an AI-generated moodboard reflecting emerging rug design styles.
- **Input example:** `"Bohemian rugs, neutral tones"`
- **Model:** Google Gemini image generation (free tier — capped at 20 generations/day)
- **Notes:** Prompts work best under 10 words, using concrete style/material keywords rather than subjective adjectives (e.g. avoid "beautiful," "nice").

### Agent 2 — Trend Discovery Agent
- **Objective:** Analyze Amazon product listings plus Pinterest, Instagram, and blog trend signals to identify emerging rug trends.
- **Input example:** `"Outdoor Rugs"` or `"Outdoor Rugs focus: Sustainable"`
- **Data sources:** Amazon (products), Pinterest, Instagram, design blogs — via backend scraping API
- **Architecture:** ~8 specialized LLM calls (trend identification, executive summary, market research, category breakdown, attribute analysis, visual trends, risk assessment, recommendations) rather than one large prompt — reduces truncation risk and improves per-section quality.
- **Models:** Mix of Llama 3.3 70B, Llama 3.2 3B, and Mistral Large across different steps.
- **Output sections:** Executive overview, market metrics, trending micro-segments, color palette, visual moodboard, attribute distribution, risk summary (with mitigation strategies), strategic recommendations.
- **Operational notes:**
  - Input parser only recognizes 4 exact category phrases (area rug, outdoor rug, hallway runner, shag rug).
  - Contains 11 built-in Wait nodes (15 sec – 2 min) to stay under free-tier rate limits on OpenRouter/Mistral Cloud — avoid running two instances simultaneously.

### Agent 3 — Competitor Monitoring Agent
- **Objective:** Benchmark Wayfair against Amazon and Walmart on pricing, ratings, and assortment; identify whitespace opportunities.
- **Input example:** `"Outdoor Rugs"` or `"Outdoor Rugs focus: Sustainable"`
- **Data sources:** Wayfair, Amazon, Walmart product listings (10 products sampled per retailer)
- **Output sections:** Executive summary, pricing comparison, ratings/reviews, side-by-side feature comparison, price band distribution, whitespace opportunities (ranked by priority), supplier/sourcing opportunities, quick-win recommendations.
- **Operational notes:**
  - Requires exact category phrase or fails with `no_category` error.
  - All figures are based on a **10-product sample per retailer** — treat as directional, not a full-catalog census.
  - Allow 3–5 minutes per run (built-in pacing to avoid rate limits).

### Agent 4 — AI Insights & Content Agent
- **Objective:** Transform the Trend and Competitor reports into on-brand marketing content — content pillars, email sequences, video scripts, ad copy, product descriptions, social captions, campaign concepts.
- **Input example:**
  - Rug Category: `Outdoor Rugs`
  - Focus Area: `Natural and Sustainable`
  - Trend Report: upload (HTML)
  - Competitor Report: upload (HTML)
  - Target Audience: `Budget-Conscious`
  - Seasonal Context: `Peak Season Launch`
- **Architecture note:** Unlike Agents 2 and 3, this uses a **single monolithic LLM call** (capped at 16,000 output tokens) to generate all content sections at once — a known source of output truncation risk on longer runs.
- **Operational notes:**
  - Keep "Focus Area" and category inputs specific — vague inputs increase truncation/incompleteness risk.
  - Don't rename the upload form fields — file matching relies on exact field names.
  - Only the first ~15,000 characters of each uploaded report are used — keep source reports concise.
  - Always check output completeness (especially later sections) before sharing.

### Dashboard Builder (Project 5)
- **Objective:** Consolidate the Trend and Competitor reports into one unified HTML dashboard for faster business decisions.
- **Architecture note:** **No AI/LLM involved.** Uses a custom code-based HTML parser (regex/string matching against known CSS classes and table structures) to extract data from the two uploaded reports, then merges it into a template fetched from the backend.
- **Why no AI here:** Merging already-structured data is a deterministic task — using an LLM would add latency, cost, and non-determinism with no accuracy benefit.
- **Known limitation:** Tightly coupled to the exact HTML structure of the upstream reports. If that structure drifts (or a report is delivered in an unexpected wrapper format), extraction fails silently rather than erroring — resulting in empty sections rather than a visible error.
- **Operational notes:**
  - Keep the category name consistent with what was used in P2/P3 — mismatches won't break parsing but will make the dashboard header inconsistent with the data shown.
  - If a section looks empty, check the source report first — this workflow only extracts what's already there.

---

## 4. Sample Key Insights (Outdoor Rugs, generated by the system)

1. **Eco-friendly rugs ($150–$250)** are a high-priority gap — no retailer in the sampled data currently offers sustainable outdoor rugs in this range.
2. **Durability & maintenance** are the top customer concerns — the only two "High" risks flagged, ahead of price or style.
3. **Premium rugs ($300+)** show as the fastest-growing segment (12.44% CAGR) with none appearing in the 10-product sample per retailer — a signal worth validating against full catalog data, not a confirmed market gap.

---

## 5. Business Value

- **Speed:** Compresses hours of manual market/competitor research into an on-demand, automated process.
- **Actionability:** Surfaces specific, prioritized gaps (e.g. an underserved price segment) rather than raw data dumps.
- **Reusability:** The same pipeline works for any product category, not just outdoor rugs — no re-hiring analysts per category.
- **Insight-to-action:** Connects research directly to draft marketing content, shortening the path from "we found an opportunity" to "we have a campaign ready to test."

---

## 6. Tech Stack

- **Orchestration:** [n8n](https://n8n.io) — visual workflow automation platform
- **LLMs:** Mistral Large, Llama 3.3 70B, Llama 3.2 3B (via Mistral Cloud & OpenRouter, free tiers)
- **Image generation:** Google Gemini (Moodboard Generator), Stable Diffusion 3 Medium via HuggingFace (Trend Agent visuals)
- **Data sources:** Backend scraping API (Amazon, Walmart, Wayfair product listings; Pinterest, Instagram, blog trend signals)
- **Output format:** Self-contained HTML reports/dashboards

---

## 7. Known Limitations & Next Steps

| Limitation | Planned Improvement |
|---|---|
| Manual file re-upload between agents | Auto-chain workflows so P2/P3 outputs feed downstream agents automatically |
| Single monolithic prompt in Agent 4 | Split into focused sub-agents, matching the Agent 2/3 pattern |
| Inconsistent retry/error handling across workflows | Standardize `retryOnFail` and try/catch coverage on all HTTP and LLM nodes |
| Data sources limited to Amazon/Pinterest/Instagram/blogs | Add Google Trends, Reddit, and Wayfair's own search/browse data for real buying-intent signals |
| Small sample sizes (10–15 products per retailer) | Pre-summarize full catalog data instead of sampling |
| Dashboard parser is brittle to upstream HTML structure changes | Add validation step that flags failed extractions instead of showing empty sections silently |
| Static, non-interactive dashboard | Add swappable templates and interactivity (filters, expandable sections, clickable charts) |
| One credential found hardcoded in a workflow (HuggingFace token) | Move all credentials to n8n's credential store; add a pre-share secret scan |
| No automated accuracy verification of AI-generated claims | Add a secondary AI verification pass that checks report claims against source data |

---

## 8. Reflections

**Key learnings:**
- Splitting one big AI task into smaller, specialized agents produces more reliable results than asking a single prompt to do everything.
- Output quality depends heavily on which model runs each step — inconsistent model selection across steps produced inconsistent quality within the same report.
- Most debugging time went into getting data to flow correctly between nodes (field names, JSON structure) — not into refining AI prompts.

**What I'd do with more time:**
- Connect all agents end-to-end automatically, removing manual handoffs.
- Diversify data sources for stronger buying-intent signals.
- Add consistent error handling across all workflows so failures are flagged, not silently incomplete.
