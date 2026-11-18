# InvoiceIQ — Agentic AI Invoice Processing (PDF → Decision) — Build Plan v2

> **For Claude Code:** Build this project phase by phase. Finish each phase completely, make sure lint, type checks and tests pass for both `frontend/` and `backend/`, then commit with a clear message before moving on. Ask me before: deploying, adding paid services, or changing the stack. Never hard-code secrets — use `.env` files and keep `.env.example` updated. Library APIs move fast (LangGraph, LangChain) — **check the current official docs before using an API**, and pin versions in `pyproject.toml` / `package.json`. When requirements are ambiguous, make a sensible assumption and log it in `docs/ASSUMPTIONS.md`.

---

## 1. Goal

An agentic process that takes a vendor invoice (PDF — digital or scanned) and produces a **clear, reasoned decision**, with every step visible live.

```
Receive & register case ─► Extract (with source evidence) ─► Normalise & validate supplier
  ─► Validate totals/tax/quality ─► Find PO + receipt ─► 2-way / 3-way match (+ contract via RAG)
  ─► Exceptions: investigate & route ─► Approval routing (approval matrix) ─► Post to ERP (mock)
  ─► Read back & verify the write ─► Archive evidence & close case
```

Every invoice is a **case**. The case keeps the original file, extracted values with source locations, records used, rules applied, match details, exceptions, approvals, ERP result and human corrections — a full decision audit trail.

**Decisions:** `APPROVED` · `NEEDS_REVIEW` · `ON_HOLD` · `REJECTED` — each with reasons, rule results (PASS/WARN/FAIL), confidence, cited policy clauses, and a suggested next action.

**Core design principle (key interview talking point):**
**The AI extracts and retrieves; the rules decide; a human approves the uncertain cases.** LLMs read invoices and contracts and write explanations. Approve/reject logic is deterministic, unit-tested Python. LangGraph pauses the graph for human review when needed.

**Context:** Fictional Indian company "Acme Retail Pvt Ltd" (Bengaluru), INR, GST invoices.

---

## 2. Why each technology is here (be able to explain this)

| Tech | Role in this project |
|---|---|
| **LangGraph** | The invoice pipeline is a `StateGraph`: one node per stage, conditional edges (e.g. scanned → vision extraction; no PO ref → PO inference), **`interrupt()` for human-in-the-loop review**, Postgres checkpointer so runs survive restarts and can resume after a reviewer decides. Also powers the "Ask AP" assistant (tool-calling agent). |
| **LangChain** | Model wrappers (`langchain-google-genai`, `langchain-groq`) with `.with_structured_output()` (Pydantic) and `.with_fallbacks()`; PDF loaders + text splitters; `PGVector` vector store + retriever. |
| **RAG** | Two uses: (1) **Contract & policy retrieval** — for each invoice, retrieve that vendor's contract clauses (agreed unit rates, payment terms) and AP policy (tolerance, approval matrix, GST rules) so checks and explanations cite the source; (2) **"Ask AP" assistant** — chat over policies and processed invoices ("Why was INV-1042 put on hold?"). |
| **FastAPI** | Python backend: REST + **Server-Sent Events** streaming of graph progress to the UI, file upload, review actions, intake webhook. Async, Pydantic v2, auto OpenAPI docs. |
| **n8n** | Business automation around the agent: **email intake** (Gmail/IMAP → FastAPI), **reviewer notifications** with deep links, vendor "missing info" emails, daily AP summary. |
| **Next.js** | Frontend: upload, live run view, dashboard, review queue, Ask-AP chat. |

---

## 2b. Production-grade behaviours (must have)

1. **Operating Procedure (editable rules, no code):** a versioned `procedure` the process owner edits in the UI (`/procedure`): tolerances (price %, quantity, tax, freight, ₹ cap), required fields, source hierarchy (PO vs contract vs GRN), approval matrix (by amount, vendor, exception type), exception owners, escalation timers, and prohibited actions. The graph loads the active version at the start of each case and records the version used. A change applies from the next case.
2. **Source evidence per field:** each extracted field stores page number + bounding box (or the text snippet it came from) + confidence, shown as highlights on the PDF preview.
3. **Touchless must be earned:** a case is auto-approved only when ALL are true — key fields ≥ confidence threshold, supplier and entity verified, matching and controls passed, no approval gate triggered, and the ERP write was confirmed by read-back. Show this checklist on every case.
4. **Exception handoff with a specific question:** every exception has a category (`PRICE_VARIANCE`, `QTY_VARIANCE`, `NO_PO`, `NO_RECEIPT`, `DUPLICATE_SUSPECT`, `BANK_CHANGE`, `TAX_MISMATCH`, `LOW_CONFIDENCE`, `OVER_PO_BALANCE`), the exact header/line variance, the policy clause, what the agent already tried, a recommendation, and **one specific question** for the named owner (e.g. "Has TechNova's rate for docking stations changed from ₹1,100 to ₹1,250? If yes, attach the amendment.").
5. **Stronger controls for sensitive actions:** bank-detail changes, overriding a tolerance/duplicate/tax flag, and approving above the matrix limit require a reason and a second approver (four-eyes). The agent can never create vendors, change bank details or release payment — it can only propose.
6. **Non-PO path:** invoices without a PO go through a separate branch: supplier legitimacy, service evidence, duplicate check, proposed GL account / cost centre / approver from rules + reviewed history (marked as a proposal with its basis), then approval. Never infer facts without evidence.
7. **ERP posting + read-back:** a **mock ERP** (own tables + API with an `ErpProvider` interface) receives the posting with normalised fields, coding, attachment, match evidence and approval history. The agent then reads the record back and verifies document number, amount, status, attachment and dimensions. Mismatch → exception. Postings are **idempotent** (idempotency key = case id).
8. **Learning from corrections:** when a reviewer corrects an extracted field or a coding proposal, store it in `corrections`. Use recent corrections for that vendor as few-shot hints in extraction prompts, and report correction rates per vendor.
9. **Metrics at every gate** (dashboard): intake (valid, duplicates, missing attachments) · extraction (field corrections by vendor) · matching (clean, within tolerance, missing records, unresolved) · exceptions (rate, owner, resolution time, top causes) · posting (successful writes, read-back mismatches) · outcome (straight-through rate, cycle time, human minutes per invoice).

---

## 3. Tech stack

### Frontend (`frontend/`)
- **Next.js** (latest stable, App Router) + **TypeScript** strict, **pnpm**
- **Tailwind CSS v4** + **shadcn/ui** + lucide-react + **Motion** (live step animations) — styled per §13b (Poppins, emerald accent)
- **TanStack Query**; native `EventSource` for SSE; Recharts (shadcn charts)
- Typed API client generated from FastAPI's OpenAPI schema (`openapi-typescript`)

### Backend (`backend/`)
- **Python 3.12+**, **uv** (package manager), **ruff** (lint/format), **pytest**, **mypy** (optional)
- **FastAPI** + Uvicorn, **Pydantic v2**, `pydantic-settings`
- **SQLAlchemy 2.0 (async)** + **asyncpg** + **Alembic** migrations
- **LangGraph** + `langgraph-checkpoint-postgres`
- **LangChain**: `langchain-core`, `langchain-google-genai`, `langchain-groq`, `langchain-postgres` (PGVector), `langchain-text-splitters`
- **PyMuPDF** (text layer + render pages to images for scanned PDFs)
- **FastEmbed** (local `BAAI/bge-small-en-v1.5`, free, no rate limits) for embeddings
- **rapidfuzz** (vendor name fuzzy matching)
- **LangSmith** (free developer plan) for tracing — optional

### Data & infra
- **Neon Postgres** (free) with **pgvector** — app data, vectors, LangGraph checkpoints
- File storage: local volume in dev; **Cloudflare R2** (free tier) in production
- **n8n** (self-hosted Docker)
- **Docker Compose** for local dev: backend, n8n (Postgres = Neon, or a local Postgres container)

### LLMs (all free tier)
| Purpose | Model |
|---|---|
| Extraction / reasoning | Gemini Flash (multimodal — reads scanned PDFs) |
| Fallback | Groq (text PDFs only) |
| Embeddings | FastEmbed (local) |

---

## 4. Cost

Vercel + Neon + Oracle Always Free VM (FastAPI + n8n) + Gemini/Groq free tiers + local embeddings = **₹0** (domain optional, ~₹200 first year for `.in`). Use only dummy data with free-tier LLM APIs.

---

## 5. Mock company data (seeded)

- **Vendors (8–10):** name, legal name, aliases, GSTIN (valid format + checksum), state code, PAN, bank account last 4, payment terms, status (`ACTIVE | BLOCKED`)
- **Purchase Orders (15–20):** number (`PO-2026-0042`), vendor, lines (description, HSN/SAC, qty, unit price, tax rate), total, status, amount already invoiced
- **Goods Receipts (GRN)** for 3-way match on some POs
- **Prior invoices** (for duplicate detection)
- **RAG corpus (`data/knowledge/`):**
  - `ap-policy.md` — tolerance rules, approval matrix (e.g. > ₹5 lakh needs Finance Manager), duplicate policy, payment-run rules
  - `gst-rules.md` — CGST/SGST vs IGST, GSTIN checks, invoice mandatory fields
  - `contracts/<vendor>.md` (or PDF) — agreed unit rates, rate validity dates, payment terms, penalty clauses — **one per vendor, some with rate changes mid-year**
  - `vendor-faq.md` — what to send when info is missing

---

## 6. LangGraph design

### State (Pydantic / TypedDict)
`run_id, file_path, file_hash, is_scanned, doc_type, extracted (InvoiceData), field_confidence, vendor, po_candidates, selected_po, retrieved_clauses[], contract_terms, rule_results[], decision, confidence, explanation, review (reviewer action/comment), errors[]`

### Nodes (one per stage; each emits progress events and writes a `run_steps` row)
1. `intake` — store file, SHA-256 hash, exact re-upload check
2. `document_check` — PyMuPDF text layer → digital vs scanned; classify doc type (invoice / credit note / quote / other)
3. `extract` — LangChain structured output (Pydantic `InvoiceData` with per-field confidence). Scanned → send page images to multimodal model. Normalise dates/amounts/GSTIN.
4. `validate` — required fields, line math, subtotal + tax = total (₹1 rounding), GSTIN checksum, intra- vs inter-state GST, date sanity
5. `vendor_match` — GSTIN exact → fuzzy name/aliases; blocked/unknown vendor; **bank details changed → fraud flag**
6. `po_match` — explicit PO ref; else infer and score candidates (vendor, items, amount)
7. `retrieve_policy` (**RAG**) — query PGVector with metadata filters (`vendor_id`, `doc_type`) for contract clauses + AP policy relevant to this invoice; LLM extracts structured `ContractTerms` (agreed unit rates, validity, payment terms) **with clause citations**
8. `duplicate_check` — same vendor + invoice no.; same vendor + amount within 30 days; same file hash
9. `tolerance_check` — invoice vs PO vs GRN vs **contract rate**; line-level price variance; partial invoicing vs PO remaining balance; approval matrix by amount
10. `decide` — **pure Python rules engine** → decision + rule results. Precedence: critical FAIL → `REJECTED`; missing dependency (no GRN, bank change) → `ON_HOLD`; any WARN or key-field confidence < 0.8 → `NEEDS_REVIEW`; else `APPROVED`
11. `route_exception` — for each exception: categorise, attach evidence + policy clause, try allowed recovery steps (e.g. re-extract a low-confidence field at higher resolution, look for a GRN posted since), then assign the named owner from the procedure and write the specific question
12. `human_review` — if exceptions remain: **`interrupt()`** with the handoff payload; graph pauses (checkpointed). Resumed via `POST /runs/{id}/review` with `Command(resume=...)`. Sensitive overrides need a second approver (second interrupt).
13. `approval_routing` — apply the approval matrix (amount, vendor, exception type) → zero, one or two approvers; each approval is another `interrupt()`; approvers see the document, coding, match result, exceptions, policy context and recommendation
14. `post_erp` — idempotent posting to the mock ERP
15. `verify_post` — read the ERP record back and compare; mismatch → exception
16. `explain` — LLM writes a plain-English summary from rule results + cited clauses (cannot change the decision)
17. `close_case` — archive evidence bundle (JSON + PDF), update metrics, trigger n8n notifications

Non-PO branch: `po_match` finds no PO → `non_po_checks` (supplier legitimacy, service evidence, propose coding/approver with basis) → `route_exception` / `approval_routing`.

### Edges
- `document_check` → `REJECTED` path if not an invoice
- `extract` → retry once with a stricter prompt if required fields missing; then continue with low confidence
- `decide` → `route_exception` → `human_review` (conditional) → `approval_routing` → `post_erp` → `verify_post` → `explain` → `close_case`
- `REJECTED` cases skip posting and go straight to `explain` → `close_case`
- Any node exception → `errors[]`, run marked `FAILED` with message (never hang)

### Streaming
FastAPI runs the graph with `astream` (`stream_mode=["updates","custom"]`) and forwards events to the browser over **SSE** (`GET /runs/{id}/events`). Persist steps too, so a page refresh reloads state.

### Ask-AP assistant (second graph)
LangGraph tool-calling agent (`create_react_agent` or custom graph) with tools:
- `search_policies(query)` — RAG over policies/contracts, returns cited chunks
- `get_run(run_id | invoice_number)` — decision, rule results, explanation
- `query_invoices(filters)` — safe, parameterised queries (by vendor, decision, date) — no free-form SQL
Answers must cite sources (`[ap-policy §3.2]`, `[INV-1042]`). Conversation memory: LangGraph Postgres checkpointer (one thread per chat session).

---

## 7. RAG details

- **Ingestion** (`backend/scripts/ingest_knowledge.py` + admin "Re-index" endpoint): load md/PDF → split by headings (`MarkdownHeaderTextSplitter`, then recursive splitter ~800 chars, 100 overlap) → metadata (`source`, `section`, `vendor_id`, `doc_type`, `valid_from/valid_to`) → embed → PGVector. Idempotent (hash per chunk).
- **Retrieval:** hybrid — PGVector similarity + Postgres full-text, merged with Reciprocal Rank Fusion; metadata filter by vendor; return top 5 with citations.
- **Grounding rules:** if retrieval finds nothing relevant (score below threshold), say so; never invent contract rates. Contract rate checks only run when a rate was retrieved with a citation; otherwise rule = `WARN: no contract rate found`.
- **RAG eval:** 15–20 question/answer pairs over the knowledge base; measure retrieval hit-rate@5 and answer correctness (LLM-as-judge with a rubric). Save to `docs/EVAL_RESULTS.md`.

---

## 8. Edge cases (4 core + 4 bonus)

| # | Edge case | Expected behaviour |
|---|---|---|
| 1 | **Scanned, low-quality invoice** | Detected as scanned → vision extraction → low-confidence fields highlighted → `NEEDS_REVIEW` → human approves → resumes → `APPROVED` |
| 2 | **Split invoicing** — PO billed across 3 invoices; 3rd exceeds PO | 1–2 `APPROVED` (remaining balance tracked); 3rd `NEEDS_REVIEW`: "exceeds PO remaining by ₹X" |
| 3 | **Price above contract rate** — unit price ₹1,250 vs contracted ₹1,100 (rate retrieved via RAG) | `NEEDS_REVIEW` with line-level variance + citation `[contracts/technova §2.1]` |
| 4 | **Duplicate / resubmitted invoice** | Exact → `REJECTED` with link to original; likely duplicate (new number, same amount) → `NEEDS_REVIEW` |
| 5 (bonus) | **Missing PO reference** | PO inferred + match score → `NEEDS_REVIEW` to confirm |
| 6 (bonus) | **Bank details changed** (fraud signal) | `ON_HOLD`: "verify with vendor via known contact before payment" (cites AP policy); release needs two approvers |
| 7 (bonus) | **Non-PO invoice** (monthly SaaS subscription) | Non-PO branch → proposed GL account + cost centre with basis ("same vendor coded to 6420 in 5 reviewed cases") → department head approval → posted |
| 8 (bonus) | **ERP read-back mismatch** (mock ERP rounds the amount) | Posting flagged, case not closed, exception "posted amount ₹1,47,499.99 ≠ approved ₹1,47,500" |

Also: non-invoice document → `REJECTED`; IGST charged on an intra-state supply → `NEEDS_REVIEW`; amount > approval limit → `NEEDS_REVIEW` routed to Finance Manager.

---

## 9. Test data & evaluation

- `backend/scripts/generate_invoices.py` (ReportLab or HTML → PDF): 25–30 invoices, 3–4 vendor layouts, all edge cases, scanned variants (render → rotate → noise → PDF), with ground-truth JSON.
- `backend/scripts/eval.py`: run all through the graph → field-level extraction accuracy, decision accuracy, avg latency, LLM calls/tokens per invoice; plus the RAG eval. Write `docs/EVAL_RESULTS.md`; show latest numbers on the dashboard.

---

## 10. Data model (SQLAlchemy + Alembic)

`vendors`, `purchase_orders`, `po_lines`, `goods_receipts`, `invoices` (extracted jsonb + key columns + file_url + file_hash), `runs` (status `QUEUED|RUNNING|WAITING_REVIEW|COMPLETED|FAILED`, decision, confidence, explanation, thread_id, durations), `run_steps`, `rule_results`, `citations` (run_id, source, section, text), `reviews`, `settings` (tolerance %, ₹ cap, confidence threshold, duplicate window), `audit_log`, plus:
- `procedures` (versioned JSON: tolerances, approval matrix, owners, escalations, prohibited actions; `active` flag) — `runs.procedure_version` records which one was used
- `field_evidence` (run_id, field, value, page, bbox, snippet, confidence)
- `exceptions` (run_id, category, variance jsonb, clause, attempted_steps, recommendation, question, owner, status, resolved_at)
- `approvals` (run_id, level, approver, role, decision, reason, decided_at, is_second_approver)
- `erp_documents` (mock ERP: doc number, normalised fields, coding, attachment url, status, idempotency_key) and `erp_postings` (run_id, request, response, readback, verified bool)
- `corrections` (run_id, vendor_id, field, old, new, reviewer, created_at) Vector tables managed by `langchain-postgres`; checkpoints by LangGraph's Postgres saver.

---

## 11. API (FastAPI)

- `POST /runs` (multipart PDF) → `{run_id}`; graph runs as a background task
- `GET /runs`, `GET /runs/{id}`, `GET /runs/{id}/events` (SSE)
- `POST /runs/{id}/review` → `{action: APPROVE|REJECT|REQUEST_INFO, comment}` → resumes graph
- `POST /runs/{id}/rerun`
- `GET /samples`, `POST /samples/{key}/run` (one-click demo inputs)
- `POST /chat` (Ask-AP, streaming)
- `GET /reference/vendors|pos|grns`, `GET/PUT /settings`, `POST /admin/reindex`, `POST /admin/reset-demo`
- `POST /intake/email` — n8n webhook, **HMAC-SHA256 signed** + timestamp check
- `GET /stats` — dashboard KPIs and gate metrics
- `GET/POST /procedures`, `POST /procedures/{version}/activate`
- `GET /approvals/inbox`, `POST /approvals/{id}/decide`
- `POST /runs/{id}/corrections` — reviewer field/coding corrections
- Mock ERP (separate router, behind `ErpProvider`): `POST /erp/documents` (idempotent), `GET /erp/documents/{doc_no}`
- `GET /cases/{id}/evidence` — evidence bundle
- CORS locked to the frontend origin; simple API key between frontend and backend

---

## 12. n8n workflows (export JSON to `n8n/workflows/`)

1. **Email intake** — Gmail/IMAP trigger (PDF attachment) → sign + POST to `/intake/email` → reply to sender "received, run #123".
2. **Review notification** — FastAPI calls n8n webhook when a run hits `WAITING_REVIEW` → email/Slack the reviewer with a deep link to `/review/{id}`; reminder after 24h if still pending.
3. **Vendor follow-up** — on `REQUEST_INFO` or missing fields → email vendor what's missing (template from `vendor-faq.md`).
4. **Daily AP summary** — scheduled → `GET /stats` → email summary (approved count/value, pending reviews, top issues).

---

## 13. UI (graded heavily — clean, intuitive, mobile-friendly, dark mode) — visual style in §13b

1. **`/` — New run:** drag-and-drop upload + "Try a sample" buttons for every edge case
2. **`/runs/[id]` — Live run view (star of the demo):** PDF preview | animated LangGraph stage timeline (pending → running → done/warn/failed, durations, expandable input/output) | extracted fields with confidence badges | rule checklist ✓⚠✗ | **cited contract/policy clauses** | decision card + explanation. When paused: "Waiting for review" banner with Approve/Reject inline.
3. **`/dashboard`:** outcome KPIs (straight-through rate, cycle time, human minutes per invoice, extraction accuracy), a **gate funnel** (intake → extraction → matching → exceptions → posting) with counts and drop-offs, exception causes and resolution time by owner, corrections by vendor, runs table with filters
4. **`/review`:** queue of paused runs; approve/reject/request info with comment
5. **`/assistant`:** Ask-AP chat with streamed answers + citation chips
6. **`/procedure`** — Operating Procedure editor (forms for tolerances, approval matrix, owners, escalations; version history with diff; "applies from next case")
7. **`/approvals`** — approver inbox (matrix-routed), with four-eyes flow for sensitive overrides
8. **`/erp`** — mock ERP ledger showing posted documents and read-back status
9. **`/cases/[id]/evidence`** — the closed case's full audit trail, downloadable as JSON + PDF
10. **`/reference`**, **`/knowledge`** (list docs, re-index), **`/settings`** (reset demo)
7. Optional: a small **graph diagram** of the LangGraph pipeline (Mermaid) with the current node highlighted

---

## 13b. Visual design (follow exactly)

Reference mockups (colors & type sheet, Dashboard, Live run view): https://claude.ai/artifact/BsxSjy2KsebEaV83SVWr4t — match their layout, spacing and colors.

**Look:** clean fintech dashboard — white cards with soft shadows on a light grey canvas, navy headings, one emerald-green accent for actions and the active place, status colors kept separate from the accent.

### Tokens (Tailwind v4 `@theme` in `frontend/app/globals.css`)

```css
@theme {
  --font-sans: "Poppins", system-ui, sans-serif;
  --font-mono: "JetBrains Mono", ui-monospace, monospace;

  /* brand */
  --color-brand-500: #10B981;   /* charts, icons, highlights */
  --color-brand-600: #059669;   /* logo gradient end */
  --color-brand-650: #04875F;   /* button gradient start */
  --color-brand-700: #047857;   /* buttons, active nav (white text) */
  --color-brand-800: #065F46;   /* text on brand tint */
  --color-brand-300: #6EE7B7;   /* evidence highlight borders, timeline line */
  --color-brand-100: #D1FAE5;   /* focus/active ring */
  --color-brand-50:  #ECFDF5;   /* soft buttons, tints */

  /* neutrals */
  --color-navy:   #0F1638;      /* headings, key numbers */
  --color-ink:    #2B3150;      /* body text */
  --color-muted:  #646E87;      /* secondary text (AA on white and canvas) */
  --color-line:   #EEF0F5;      /* dividers, table rows */
  --color-canvas: #F7F8FB;      /* page background, inputs */
  --color-card:   #FFFFFF;

  /* decision status (text / fill) */
  --color-approved: #15803D;  --color-approved-bg: #DCFCE7;
  --color-review:   #B45309;  --color-review-bg:   #FEF3C7;
  --color-hold:     #1D4ED8;  --color-hold-bg:     #DBEAFE;
  --color-rejected: #B91C1C;  --color-rejected-bg: #FEE2E2;

  --radius-card: 18px;
  --radius-control: 12px;
  --shadow-card: 0 8px 28px rgba(15, 22, 56, 0.06);
  --shadow-brand: 0 6px 16px rgba(5, 150, 105, 0.28);
}
```

**Dark mode:** background `#0F1426`, card `#171D33`, border `#262E48`, text `#E7EAF3`, muted `#9AA3B8`, accent `#34D399`. Status fills become 15% opacity of the status color; status text uses the 400 shade.

**Fonts:** load Poppins (400/500/600/700) and JetBrains Mono (400/500) with `next/font/google`. Use mono for invoice numbers, PO numbers, GSTINs and citations.

**Type scale:** page title 28/36 semibold navy · card title 18/26 semibold navy · body 14/22 · label 12/18 muted · big numbers 30–34 bold navy.

### Components (shadcn/ui, restyled)
- **Card:** white, `rounded-[18px]`, `shadow-card`, no border, padding 22–24px.
- **Primary button:** `bg-gradient-to-br from-brand-650 to-brand-700 text-white shadow-brand rounded-[12px] h-11 px-5 font-semibold`.
- **Soft button:** `bg-brand-50 text-brand-700`. **Danger soft:** `bg-rejected-bg text-rejected`.
- **Sidebar (248px):** logo (green gradient tile + "InvoiceIQ") with a collapse button; groups with small uppercase labels **WORKFLOW** (Dashboard, New run, Review queue, Approvals, ERP ledger), **SETUP** (Procedure, Knowledge base, Vendors & POs), **HELP** (Ask AP, Settings); each item = icon + label + optional count pill + chevron; active item uses the primary button style. Collapses to icons on tablet; becomes a drawer on mobile.
- **Top bar:** search input on canvas background, notification bell with green dot, messages, avatar + name + role, light/dark toggle.
- **Status badge:** pill, status fill + status text + icon (check / alert / pause / x) — never color alone.
- **Confidence chip:** ≥ 0.8 → approved colors; < 0.8 → review colors.
- **Citation chip:** mono 12px on canvas background, e.g. `[contracts/technova §2.1]`.
- **Segmented control** (1D 7D 1M 3M ALL): canvas track; active segment uses the primary gradient.
- **Charts (Recharts):** brand-500 line, 2.5px, area fill brand-500 from 22% → 0% opacity, white tooltip card with shadow, no gridlines except a faint baseline.
- **Pipeline timeline:** 24px circular step dots — done (approved colors + check), warning (review colors), current/paused (green gradient + `brand-100` ring, pulsing while running), pending (dashed outline). Connector line brand-300 for completed steps, line color for pending. Animate state changes with Motion.
- **Evidence highlights on the PDF preview:** brand-50 fill + brand-300 border; variance fields use review colors.
- **Decision card:** 4px top border in the decision's status color; "Question for you" box on `#FFFBEB` when waiting for review.
- **Icons:** lucide-react, 1.9px stroke, 18px in nav.

### Accessibility
White text only on brand-700/650 (never on brand-500). Muted text never lighter than `#646E87`. Touch targets ≥ 44px. Every icon-only button has an `aria-label`. Status always shown with icon + text.

---

## 14. Repo structure

```
/frontend                      # Next.js app
/backend
  /app
    main.py, config.py, deps.py
    /api (runs.py, review.py, chat.py, intake.py, reference.py, admin.py, stats.py)
    /graph (state.py, nodes/*.py, builder.py, assistant.py)
    /rules (math.py, gst.py, tolerance.py, duplicates.py, decide.py)   # pure, unit-tested
    /llm (models.py — provider switch + fallbacks, prompts.py, schemas.py)
    /rag (ingest.py, retriever.py)
    /db (models.py, session.py) + /alembic
  /scripts (generate_invoices.py, ingest_knowledge.py, eval.py, seed.py)
  /tests
/data (seed json/csv, knowledge/, test-invoices/)
/n8n/workflows/*.json
/deploy (docker-compose.yml, Caddyfile, DEPLOY.md)
/docs (ARCHITECTURE.md, ASSUMPTIONS.md, EVAL_RESULTS.md)
```

---

## 15. Environment variables (`backend/.env.example`)

```
APP_ENV=local
DATABASE_URL=
GOOGLE_API_KEY=
GROQ_API_KEY=
LANGSMITH_API_KEY=               # optional
LANGSMITH_TRACING=false
STORAGE_BACKEND=local            # local | r2
R2_ACCOUNT_ID=                   # r2 only
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=
R2_BUCKET=
N8N_BASE_URL=
N8N_WEBHOOK_SECRET=
FRONTEND_ORIGIN=http://localhost:3000
API_KEY=
```
Frontend: `NEXT_PUBLIC_API_URL`, `API_KEY` (server-side only).

---

## 16. Build phases (~3 weeks)

### Phase 1 — Foundation (Days 1–2)
Monorepo, Next.js app shell built to §13b (theme tokens, fonts, sidebar, top bar, card/button/badge components, dark mode), FastAPI skeleton with health check, uv/ruff/pytest, Alembic + Neon + pgvector, seed data, `/reference` page, Docker Compose for local dev.
**Done when:** frontend shows vendors/POs fetched from FastAPI.

### Phase 2 — Test data & knowledge base (Day 3)
Generate invoices + scanned variants + ground truth; write policy, GST, contract and FAQ docs.
**Done when:** ~25 realistic PDFs and the knowledge docs exist.

### Phase 3 — Extraction with LangChain (Days 4–5)
Provider switch + fallbacks, Pydantic schemas, digital vs scanned handling, normalisation.
**Done when:** happy-path and scanned invoices extract with confidence scores (pytest with recorded fixtures).

### Phase 4 — Rules engine (Day 6)
Pure-Python validation, GST, vendor/PO match, duplicates, tolerance, partial invoicing, approval matrix, decision precedence — full unit tests.
**Done when:** every edge case yields the expected decision from fixture data.

### Phase 5 — RAG (Days 7–8)
Ingestion, hybrid retrieval, `retrieve_policy` node with citations, contract-rate extraction, RAG eval set.
**Done when:** edge case #3 flags the price with a correct contract citation.

### Phase 6 — LangGraph pipeline + streaming (Days 9–10)
Assemble the graph, Postgres checkpointer, `interrupt()` human review + resume, SSE events, background execution.
**Done when:** a scanned invoice pauses for review, and approving it resumes the graph to completion.

### Phase 7 — Live run view, review queue, dashboard (Days 11–13)
All UI pages from §13, sample buttons, citations, charts.
**Done when:** every sample runs live in the UI under ~30 seconds, and review works end-to-end.

### Phase 8 — Procedure, approvals, mock ERP & case closure (Days 14–15)
Versioned Operating Procedure + editor; exception routing with specific questions; approval matrix + four-eyes overrides; non-PO branch; mock ERP with idempotent posting + read-back verification; corrections feeding extraction hints; evidence bundle; gate metrics.
**Done when:** changing the price tolerance in `/procedure` changes the next case's decision; an approved invoice is posted and verified in `/erp`; the closed case shows its full audit trail.

### Phase 9 — Ask-AP assistant (Day 16)
Tool-calling agent with RAG + run lookup, streaming chat UI with citations.
**Done when:** "Why was the TechNova invoice flagged?" returns the right reason with citations.

### Phase 10 — n8n automations (Day 17)
Four workflows, HMAC-signed webhooks, export JSON.
**Done when:** emailing a PDF creates a run, and the reviewer gets a notification with a deep link.

### Phase 11 — Evaluation (Day 18)
Run `eval.py`, tune prompts/rules, document changes and results.
**Done when:** `docs/EVAL_RESULTS.md` shows extraction, decision and RAG metrics.

### Phase 12 — Polish, docs & deploy (Days 19–21)
Empty/loading/error states, mobile layout, reset-demo; README (problem, architecture diagram in Mermaid, LangGraph diagram, "why each tool" table, edge cases, eval results, setup, what's next); Playwright e2e; deploy plan: frontend → Vercel, FastAPI + n8n + Caddy → Oracle Always Free VM (ARM64 images), DB → Neon. **Ask me before deploying.**
**Done when:** build/lint/tests pass, deploy guide complete.
