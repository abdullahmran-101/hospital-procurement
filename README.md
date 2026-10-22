# Hospital Procurement & Inventory Intelligence — V2

V1's loop: **shortage detected → AI recommendation → human approval → purchase order.**
V2 adds real demand forecasting — `avg_daily_usage` is no longer a static seeded number,
it's a rolling average computed from `usage_logs` before every shortage check.

## Architecture (V2)

```
n8n (scheduled, 8am daily)
   │
   ▼
POST /forecasting/update ────► recalculates avg_daily_usage from usage_logs (7-day rolling avg)
   │
   ▼
GET /shortage/check ─────────► Postgres (shortage_check view, now using fresh averages)
   │
   ▼ (per item at risk)
POST /recommendations/generate ─► picks cheapest supplier + LLM writes reasoning (Groq)
   │
   ▼
POST /purchase-orders/  ─────► creates PO with status=pending_approval
   │
   ▼
Notify Manager (real email node) ──► manager clicks Approve/Reject
   │
   ▼
GET webhook /po-decision ────► POST /purchase-orders/{id}/decision
```

## What V2 adds over V1

- **`usage_logs` table** — one row per item per day of actual consumption.
  Seeded with 30 days of noised history (±15% around the old static baseline)
  so the moving average has something realistic to work with.
- **`POST /forecasting/update`** — recomputes `avg_daily_usage` as a simple
  moving average over a configurable window (default 7 days). Called first
  in the n8n workflow, every run, before the shortage check.
- **`POST /forecasting/log-usage`** — the entry point for recording real
  consumption going forward (pharmacy/OR system would call this daily;
  for now it's manual or n8n-fed).
- **Supplier scoring from real order history** — `purchase_orders` now tracks
  `expected_delivery_date` (set at approval, from the item's lead time) and
  `delivered_at` (set via `PATCH /suppliers/purchase-orders/{id}/deliver`).
  `POST /suppliers/recalculate-scores` computes each supplier's
  `reliability_score` as 60% on-time delivery rate + 40% price competitiveness,
  and only overwrites a supplier's score once it has ≥3 delivered orders —
  before that, the manually seeded default holds.
  Seeded with 11 historical delivered orders across the 3 suppliers so scoring
  works immediately: MedSupply Lahore is reliably on-time, National Surgical
  Traders is cheap but often late, PharmaLink is pricier but always on time.
- **Supplier pick is now a composite score** (50% price competitiveness,
  50% reliability), not just cheapest-price — see `_pick_supplier` in
  `ai_recommendation.py`.
- **New n8n workflow** — `4-weekly-supplier-scoring.json` runs
  `/suppliers/recalculate-scores` every Monday 6am.

V2 is now feature-complete per the original plan's list. Both forecasting and
supplier scoring are deliberately simple (moving average, not a trend model;
a linear weighted score, not a learned ranking) — that's the honest baseline
to build on, not a placeholder to be embarrassed about.

## Run it

1. Copy `backend/.env.example` to `backend/.env` and add your `GROQ_API_KEY`
   (or swap the client in `app/services/ai_recommendation.py` for OpenAI/Gemini/OpenRouter).
2. `docker compose up --build`
   - Postgres on `:5432` (auto-seeded from `db/schema.sql` + `db/seed.sql`)
   - FastAPI on `:8000` (docs at `http://localhost:8000/docs`)
   - n8n on `:5678` (login: `admin` / `changeme`, set in `docker-compose.yml`)
3. In the n8n UI, import all four workflows from `n8n/workflows/`:
   - `1-shortage-to-po.json` — the daily forecast + shortage + recommendation automation
   - `2-approval-webhook.json` — the approval callback
   - `3-error-notifier.json` — catches any failure across the automation and emails you
   - `4-weekly-supplier-scoring.json` — recalculates supplier reliability weekly
4. Set up an SMTP credential in n8n (Settings → Credentials → SMTP — Gmail app password
   works fine for testing), then point both `Notify Manager` (workflow 1) and
   `Email Alert` (workflow 3) at it — replace `REPLACE_WITH_YOUR_SMTP_CREDENTIAL_ID`.
5. After importing workflow 3, copy its workflow ID from the n8n URL bar and paste it
   into workflow 1's `settings.errorWorkflow` field (Workflow Settings panel) so failures
   in the main automation route to the error notifier.
6. Trigger workflow 1 manually first (don't wait for 8am) to test the full loop.
   With the seed data, "Surgical Gloves" should be flagged immediately —
   it matches the shortage example from the original plan (~6 days remaining).

## Reliability built into V1

- Every `HTTP Request` node in workflow 1 (`Check Shortages`, `Generate AI Recommendation`,
  `Create Pending PO`) retries 3x with a delay before failing — covers transient network
  blips and Groq rate limits without you babysitting it.
- If a node still fails after retries, n8n's workflow-level `errorWorkflow` setting routes
  the failure to `3-error-notifier.json`, which emails you the failed node, the error
  message, and a link to the execution so you can re-run it.
- The `Notify Manager` node is a real `emailSend` node (not a placeholder) — it renders
  the item, recommended supplier, price, and AI reasoning, with Approve/Reject links
  that hit workflow 2's webhook directly.

## Verify the loop manually (without n8n)

```bash
curl -X POST "http://localhost:8000/forecasting/update?window_days=7"
curl http://localhost:8000/shortage/check?threshold_days=7
curl -X POST http://localhost:8000/recommendations/generate -H "Content-Type: application/json" -d '{"item_id": 1}'
curl -X POST http://localhost:8000/purchase-orders/ -H "Content-Type: application/json" -d '{"item_id":1,"supplier_id":2,"quantity":300,"unit_price":790,"ai_reasoning":"test"}'
curl -X POST http://localhost:8000/purchase-orders/1/decision -H "Content-Type: application/json" -d '{"decision":"approved","approved_by":"Abdullah"}'
curl http://localhost:8000/purchase-orders/
curl -X PATCH http://localhost:8000/suppliers/purchase-orders/1/deliver
curl -X POST http://localhost:8000/suppliers/recalculate-scores
```

## What's deliberately NOT in V1

- No demand forecasting model (avg_daily_usage is static/manual for now)
- No supplier scoring beyond price + a manually seeded reliability number
- No invoice/document parsing
- No frontend dashboard
- No auth/RBAC/multi-tenancy

## Roadmap

- **V2 (this version)** — Real demand forecasting via moving average ✅.
  Supplier scoring from historical PO data ✅.
- **V3** — Document automation: invoice/PO parsing (image/PDF → structured data),
  delivery note matching
- **V4** — Next.js dashboard, auth, proper multi-agent orchestration
  (Procurement / Supplier Research / Document / Inventory / Finance agents)
- **V5** — Multi-tenancy, RBAC, production deployment, monitoring
