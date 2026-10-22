# Hospital Procurement & Inventory Intelligence

An automated procurement loop for hospital inventory: it forecasts demand from real consumption data, detects impending stock-outs, selects the best supplier using a composite score, drafts the purchase order with an LLM-written rationale, and routes it to a human for approval. Nothing is ordered without a manager's explicit decision.

**Current version: V2** (demand forecasting and data-driven supplier scoring)

---

## Table of Contents

1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Core Capabilities](#core-capabilities)
4. [Design Decisions](#design-decisions)
5. [Data Model Notes](#data-model-notes)
6. [API Reference (Summary)](#api-reference-summary)
7. [Getting Started](#getting-started)
8. [Verifying the Loop Without n8n](#verifying-the-loop-without-n8n)
9. [Reliability and Failure Handling](#reliability-and-failure-handling)
10. [Known Limitations](#known-limitations)
11. [Roadmap](#roadmap)

---

## Overview

Stock-outs of critical supplies such as surgical gloves, medications, and consumables carry direct clinical and financial cost, yet procurement in many hospitals still depends on manual stock checks and static reorder points.

This system closes that loop end to end:

```
shortage detected -> AI recommendation -> human approval -> purchase order
```

**V1** established the loop using a static, manually seeded `avg_daily_usage`.
**V2** makes the loop data-driven: demand is now forecast from actual consumption history, and supplier selection is informed by measured delivery performance rather than a hand-entered number.

The guiding principle is **AI proposes, humans decide**. The LLM contributes reasoning and explanation; all ranking, forecasting, and scoring logic is deterministic, inspectable, and testable.

---

## System Architecture

```
n8n (scheduled, 08:00 daily)
   |
   v
POST /forecasting/update ---------> Recompute avg_daily_usage from usage_logs
   |                                (7-day simple moving average, configurable)
   v
GET  /shortage/check -------------> Postgres `shortage_check` view
   |                                (evaluates days-of-stock-remaining on fresh averages)
   v   (per item at risk)
POST /recommendations/generate ---> Composite supplier selection
   |                                + LLM-authored reasoning (Groq)
   v
POST /purchase-orders/ -----------> PO created, status = pending_approval
   |
   v
Notify Manager (SMTP email) ------> Manager clicks Approve / Reject
   |
   v
GET  webhook /po-decision --------> POST /purchase-orders/{id}/decision
                                    (sets expected_delivery_date on approval)

n8n (scheduled, Mondays 06:00)
   |
   v
POST /suppliers/recalculate-scores -> Reliability scores refreshed from order history
```

**Stack:** FastAPI, PostgreSQL, n8n, Groq (swappable LLM client), Docker Compose.

**Separation of concerns:** n8n owns orchestration, scheduling, retries, and notification. FastAPI owns business logic and persistence. Postgres owns the source of truth, including the shortage view. The LLM is confined to generating human-readable reasoning.

---

## Core Capabilities

### 1. Demand forecasting

- `usage_logs` stores one row per item per day of actual consumption.
- `POST /forecasting/update` recomputes `avg_daily_usage` as a simple moving average over a configurable window (default 7 days). It runs first in every daily workflow, so every shortage check operates on current data.
- `POST /forecasting/log-usage` is the ingestion point for real consumption. In production, a pharmacy or operating-room system would call it daily; for now it can be invoked manually or fed by n8n.
- The database is seeded with 30 days of history, noised +/-15% around the former static baseline, so the moving average behaves realistically from the first run.

### 2. Shortage detection

The `shortage_check` Postgres view computes days of stock remaining per item from current inventory and the freshly updated average. Items below the configured threshold (default 7 days) are passed downstream. With the seed data, Surgical Gloves is flagged immediately at roughly 6 days remaining.

### 3. Supplier scoring from order history

`purchase_orders` now records the full delivery lifecycle:

| Field | Set when |
|---|---|
| `expected_delivery_date` | At approval, derived from the item's lead time |
| `delivered_at` | Via `PATCH /suppliers/purchase-orders/{id}/deliver` |

`POST /suppliers/recalculate-scores` derives each supplier's `reliability_score`:

```
reliability_score = 0.60 * on_time_delivery_rate + 0.40 * price_competitiveness
```

A supplier's score is only overwritten once it has **at least 3 delivered orders**; below that threshold, the manually seeded default is retained. This prevents a single late shipment from distorting a supplier's ranking.

Seed data includes 11 historical delivered orders so scoring is meaningful immediately:

- **MedSupply Lahore:** consistently on time
- **National Surgical Traders:** low price, frequently late
- **PharmaLink:** higher price, always on time

### 4. Composite supplier selection

`_pick_supplier` in `app/services/ai_recommendation.py` ranks candidates by:

```
selection_score = 0.50 * price_competitiveness + 0.50 * reliability_score
```

This replaces V1's cheapest-price rule. A cheap supplier that is routinely late no longer wins by default, because late delivery during a shortage is the failure this system exists to prevent.

### 5. Human-in-the-loop approval

Each PO is created as `pending_approval`. The manager receives an email containing the item, recommended supplier, unit price, quantity, and the AI's reasoning, with one-click Approve and Reject links that call back into n8n's approval webhook.

---

## Design Decisions

**Deterministic core, LLM at the edge.** Forecasting, scoring, and supplier ranking are plain arithmetic. The LLM only explains a decision that has already been made by auditable logic. This keeps behavior reproducible and removes model variance from procurement-critical choices.

**Simple moving average as a deliberate baseline.** A 7-day SMA is transparent, cheap, and easy to reason about. It gives the pipeline real data flow now, and it establishes a benchmark any future model (trend, seasonality, ML-based) must beat to justify its complexity.

**Minimum-sample guard on scoring.** The 3-delivery threshold trades responsiveness for stability. Early scores reflect priors, not noise.

**Orchestration in n8n, logic in the API.** Business rules stay in versioned, testable Python; scheduling, retries, and notifications stay in a visual workflow tool that operators can inspect and modify without redeploying the backend.

**Provider-agnostic LLM layer.** The Groq client can be swapped for OpenAI, Gemini, or OpenRouter by editing a single module (`app/services/ai_recommendation.py`).

---

## Data Model Notes

| Table / View | Purpose |
|---|---|
| `usage_logs` | Daily per-item consumption; source for the moving average |
| `shortage_check` (view) | Days of stock remaining per item, using current `avg_daily_usage` |
| `purchase_orders` | PO lifecycle: `pending_approval`, decision, `expected_delivery_date`, `delivered_at` |
| `suppliers` | Supplier records including `reliability_score` |

Schema and seed data live in `db/schema.sql` and `db/seed.sql` and are applied automatically on first container start.

---

## API Reference (Summary)

Interactive documentation is served at `http://localhost:8000/docs`.

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/forecasting/update?window_days=7` | Recompute `avg_daily_usage` from `usage_logs` |
| POST | `/forecasting/log-usage` | Record actual daily consumption |
| GET | `/shortage/check?threshold_days=7` | List items projected to run short |
| POST | `/recommendations/generate` | Select supplier and generate AI reasoning for an item |
| POST | `/purchase-orders/` | Create a PO in `pending_approval` |
| GET | `/purchase-orders/` | List purchase orders |
| POST | `/purchase-orders/{id}/decision` | Approve or reject a PO |
| PATCH | `/suppliers/purchase-orders/{id}/deliver` | Mark a PO delivered (feeds scoring) |
| POST | `/suppliers/recalculate-scores` | Recompute supplier reliability scores |

---

## Getting Started

### Prerequisites

- Docker and Docker Compose
- A Groq API key (or credentials for an alternative LLM provider)
- An SMTP account for notifications (a Gmail app password is sufficient for testing)

### Setup

1. **Configure environment.** Copy `backend/.env.example` to `backend/.env` and set `GROQ_API_KEY`. To use a different provider, replace the client in `app/services/ai_recommendation.py`.

2. **Start the stack.**
   ```bash
   docker compose up --build
   ```
   | Service | Address | Notes |
   |---|---|---|
   | PostgreSQL | `:5432` | Auto-seeded from `db/schema.sql` and `db/seed.sql` |
   | FastAPI | `:8000` | Docs at `/docs` |
   | n8n | `:5678` | Default login `admin` / `changeme` (set in `docker-compose.yml`) |

3. **Import the n8n workflows** from `n8n/workflows/`:

   | Workflow | Role |
   |---|---|
   | `1-shortage-to-po.json` | Daily forecast, shortage check, recommendation, PO creation, manager notification |
   | `2-approval-webhook.json` | Handles Approve / Reject callbacks |
   | `3-error-notifier.json` | Emails failure details for any workflow error |
   | `4-weekly-supplier-scoring.json` | Recalculates supplier reliability every Monday 06:00 |

4. **Configure SMTP.** Create an SMTP credential in n8n (Settings, Credentials, SMTP), then attach it to both the `Notify Manager` node (workflow 1) and the `Email Alert` node (workflow 3), replacing `REPLACE_WITH_YOUR_SMTP_CREDENTIAL_ID`.

5. **Wire the error handler.** After importing workflow 3, copy its workflow ID from the n8n URL and set it as `settings.errorWorkflow` in workflow 1's Workflow Settings panel.

6. **Run an end-to-end test.** Trigger workflow 1 manually rather than waiting for 08:00. With the seed data, Surgical Gloves should be flagged immediately, matching the shortage scenario from the original project plan.

---

## Verifying the Loop Without n8n

Each stage can be exercised directly against the API:

```bash
# 1. Refresh demand estimates
curl -X POST "http://localhost:8000/forecasting/update?window_days=7"

# 2. Detect at-risk items
curl "http://localhost:8000/shortage/check?threshold_days=7"

# 3. Generate a recommendation
curl -X POST http://localhost:8000/recommendations/generate \
  -H "Content-Type: application/json" -d '{"item_id": 1}'

# 4. Create a pending PO
curl -X POST http://localhost:8000/purchase-orders/ \
  -H "Content-Type: application/json" \
  -d '{"item_id":1,"supplier_id":2,"quantity":300,"unit_price":790,"ai_reasoning":"test"}'

# 5. Approve it
curl -X POST http://localhost:8000/purchase-orders/1/decision \
  -H "Content-Type: application/json" \
  -d '{"decision":"approved","approved_by":"Abdullah"}'

# 6. Inspect orders
curl http://localhost:8000/purchase-orders/

# 7. Record delivery, then refresh supplier scores
curl -X PATCH http://localhost:8000/suppliers/purchase-orders/1/deliver
curl -X POST http://localhost:8000/suppliers/recalculate-scores
```

---

## Reliability and Failure Handling

- **Automatic retries.** The `Check Shortages`, `Generate AI Recommendation`, and `Create Pending PO` HTTP nodes in workflow 1 each retry three times with a delay, absorbing transient network faults and LLM rate limits without operator involvement.
- **Centralized error routing.** If a node still fails after retries, n8n's workflow-level `errorWorkflow` hands the failure to `3-error-notifier.json`, which emails the failed node, the error message, and a direct link to the execution for re-run.
- **Real notification path.** `Notify Manager` is a production `emailSend` node, not a placeholder. It renders item, supplier, price, and AI reasoning, with Approve/Reject links bound to workflow 2's webhook.
- **Fresh-data guarantee.** Forecast recomputation is the first step of every run, so a shortage decision never rests on a stale average.

---

## Known Limitations

These are conscious scope boundaries, not oversights:

- **Forecasting is a simple moving average.** It lags real trend changes and does not model seasonality, outbreaks, or scheduled procedures. It is the baseline for later models to beat.
- **Supplier scoring is a linear weighted score**, not a learned ranking. Weights (60/40 for reliability, 50/50 for selection) are fixed and not yet tuned against outcomes.
- **No document ingestion.** Invoices, delivery notes, and POs are not yet parsed from images or PDFs.
- **No frontend.** Interaction is via API, n8n, and email.
- **No authentication, RBAC, or multi-tenancy.** Treat the current deployment as a trusted-network prototype, and change the default n8n credentials (`admin` / `changeme`) before exposing it anywhere.

---

## Roadmap

| Version | Focus | Status |
|---|---|---|
| V1 | Shortage-to-PO loop with human approval | Complete |
| V2 | Rolling-average demand forecasting; supplier scoring from real order history | Complete (this version) |
| V3 | Document automation: invoice and PO parsing (image/PDF to structured data), delivery-note matching | Planned |
| V4 | Next.js dashboard, authentication, multi-agent orchestration (Procurement, Supplier Research, Document, Inventory, Finance) | Planned |
| V5 | Multi-tenancy, RBAC, production deployment, monitoring | Planned |
