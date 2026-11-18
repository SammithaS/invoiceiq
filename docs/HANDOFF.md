# Handoff — where the build stands

Read this first in a new session, then `docs/BUILD_PLAN.md` (the full plan) and
`docs/ASSUMPTIONS.md` (every decision so far, #1–98).

## Status

- Repo `C:\Users\asus\Desktop\invoiceiq`, branch `main`. Phases 1–10 committed (Phase 10's n8n side still needs the user's
  n8n cloud account — see below).
- **Next: Phase 11 — Evaluation.**
- Working agreement: finish a phase fully; ruff + mypy + pytest pass; commit per phase on `main`;
  log ambiguities in `docs/ASSUMPTIONS.md`; ask before deploying, paid services or stack changes.
  The user puts API keys in `backend/.env` themselves.

## Environment (Windows)

- Backend commands run from `backend/` as `python -m uv run ...` (`uv` is not on PATH), with
  `UV_PROJECT_ENVIRONMENT=C:/Users/asus/.venvs/invoiceiq` and `PYTHONIOENCODING=utf-8`.
- Checks: `python -m uv run ruff check .`, `ruff format --check .`, `mypy app scripts tests`,
  `pytest -q` (522 passing at the end of Phase 10).
- Neon DB in `backend/.env` (`DATABASE_URL`); `GOOGLE_API_KEY` and `GROQ_API_KEY` are set.
- Dev servers may already be running (backend :8000, frontend :3000) — check ports first.
  The backend must run with `--loop app.loop:selector_loop_factory` on Windows (psycopg).
  Frontend checks: `pnpm typecheck`, `pnpm lint` and `pnpm build` (the build catches
  cacheComponents/Suspense errors that dev doesn't). `.claude/launch.json` has
  `backend-live` (:8010) and `frontend-live` (`next start` on :3010, API_URL → :8010) so live
  checks never touch the user's own :8000/:3000 servers.

## What exists

- **Phase 1:** FastAPI + async SQLAlchemy + Alembic on Neon; seed `data/seed/*.json` (10 vendors,
  20 POs, 17 GRNs, 9 prior invoices) built by `scripts/build_seed_data.py`, loaded by
  `scripts/seed.py`; Next.js shell + `/reference`; `app/rules/gst.py` (GSTIN checksum, state codes).
- **Phase 2:** 25 test PDFs in `data/test-invoices/` with `ground-truth/<key>.json`
  (`fields`; `expected` = decision, exceptions, approvals, final_decision, reasons, citations,
  matched_po, matched_grn, proposed_coding; `depends_on`; `erp_fault`) and `manifest.json` in run
  order. Specs: `backend/scripts/invoice_specs.py`. Knowledge base `data/knowledge/`
  (`ap-policy.md`, `gst-rules.md`, `vendor-faq.md`, `contracts/<vendor>.md`) with numbered
  sections for citations.
- **Phase 3:** `app/extraction/` (document, normalise, evidence, extractor, models) and `app/llm/`
  (schemas, prompts, models). `extract_invoice(pdf) -> ExtractionResult` with `data: InvoiceData`
  (Decimal amounts, dates, lines), per-field `evidence`, `low_confidence`, `missing_required`,
  `is_scanned`, `scan_quality`, `model`, tokens. 100% field accuracy on all 25 (recorded with the
  Gemini flash-lite fallback; primary was overloaded). Fixtures in
  `backend/tests/fixtures/extraction/`; `scripts/eval_extraction.py` (`--record` / `--replay`)
  provides `compare()` and `replay_factory(key)`.
- **Phase 4:** `app/rules/` — `evaluate(extraction, ref, ctx, policy) -> Assessment` (decision,
  PASS/WARN/FAIL/SKIP rule results with citations, exceptions with owner + specific question,
  approvals, matched PO/GRN, proposed coding, touchless checklist) and `history_record()` to
  register a case for later ones. `Policy` holds the #17 numbers; `load_seed_reference()` builds
  `ReferenceData` from the seed JSON. `tests/test_rules_cases.py` runs all 25 cases in manifest
  order (all match ground truth); `tests/contract_fixture.py` parses the contract Markdown into
  `ContractTerms` as a stand-in for RAG. Design decisions: ASSUMPTIONS #31–42.

- **Phase 5:** `app/rag/` — `ingest` (Markdown → 191 citable chunks, content-hash ids),
  `store` (`PgKnowledgeStore` on Neon via `langchain-postgres` `PGVectorStore`, table
  `knowledge_chunks`; `MemoryKnowledgeStore` for tests), `retriever` (`HybridRetriever`: dense +
  OR-ed full-text, RRF, relevance gate, AP glossary), `index` (`reindex`, idempotent),
  `contract_terms` (`contract_terms_for_invoice` = retrieve vendor clauses → LLM reads
  `ContractTermsRead` → `ground()` keeps only facts the cited excerpt states → `ContractTerms`).
  `app.llm.models.structured_model(schema)` is the generic structured-output call. API:
  `GET /knowledge`, `POST /knowledge/search`, `POST /admin/reindex`. Scripts:
  `ingest_knowledge`, `eval_rag` (`--pg`, `--answers`, `--write`, `--record-contracts`,
  `--case <key>`). Neon is indexed. Contract readings recorded in
  `tests/fixtures/contract_terms/` and replayed by `test_rules_cases.py`.

- **Phase 6:** `app/graph/` — `state` (JSON-only `CaseState`), `nodes` (one per stage, `step()`
  wrapper: events, timings, error capture; `human_review` = `interrupt()`), `builder`
  (`build_graph(deps, checkpointer)`), `deps` (`GraphDeps`: extract / contract terms / explain /
  policy / today — tests pass replay versions), `explain` (LLM summary + template fallback),
  `repo` (reference data from the DB, invoice-history upserts), `runner` (`RunService`: background
  tasks, `run_steps`, SSE broker, `review()`, `recover()`; `open_checkpointer`). API in
  `app/api/runs.py`. Migration 0002 applied on Neon. Live-checked on Neon, including a restart
  while paused (ASSUMPTIONS #62). Launch config `backend-live` runs on :8010.

- **Phase 7:** UI on the runs API. `/` upload + 25 sample cards; `/runs/[id]` live view
  (`lib/use-run.ts`: EventSource + snapshot refetch; `app/runs/[id]/`: timeline, PDF page images
  with evidence boxes from `GET /runs/{id}/pages/{n}`, decision card with inline review, touchless
  checklist, cited sources, extracted fields, rule checks); `/review` queue
  (`components/review-form.tsx` shared); `/dashboard` on `GET /stats`; sidebar review count.
  Backend speed work (#63, #64, #71, #72) and `scripts/run_samples_live.py` (timing pass; reseed
  first). Results: ASSUMPTIONS #73.

- **Phase 8:** versioned procedure (`app/procedure.py`, `app/api/procedures.py`, `/procedure`);
  graph nodes `route_exception`, `approval_routing` (one interrupt per approver, four-eyes by
  name), `post_erp`, `verify_post` (`app/erp.py`: `ErpProvider`, `MockErp`, `compare`);
  `app/graph/case_repo.py` (exceptions, approvals, postings, correction hints);
  `app/evidence.py` (JSON + PDF bundle); `app/api/case.py` (records, approvals inbox/decide,
  corrections, evidence, ERP ledger/correct, `POST /admin/reset-demo`). Migration 0003 applied on
  Neon. UI: `/approvals`, `/erp`, `/cases/[id]/evidence`, `/settings` (reset demo), run page
  exception/approval/ERP cards and field corrections, dashboard ERP/owner/correction cards.
  Procedure v1 is active on Neon (v2 with 1% tolerance exists from the live check). ASSUMPTIONS
  #74–83.

- **Phase 9:** Ask AP. `app/assistant/` — `tools` (`AssistantTools`: `search_policies`,
  `find_cases`, `get_case`, `case_stats`; read-only, sources as tool artifacts), `agent`
  (`build_assistant(tools, model_factory, checkpointer)`: agent ⇄ tools → finalize, 5-round cap,
  citation check, tool-dump guard), `service` (`AssistantService.ask()` → chat events, `history()`).
  `app.llm.models.chat_model(tools)` = tool-calling model with the provider fallbacks. API
  `app/api/chat.py`: `POST /chat` (SSE), `GET /chat/{thread_id}`; wired in `app/main.py` on the
  shared checkpointer (off with a warning if no LLM key). UI `/assistant` (`app/assistant/ask-ap.tsx`,
  `lib/chat.ts`): suggestions, live tool steps, streamed answer, citation chips (case → run page,
  clause → its excerpt), sources list, new chat, restore on reload. ASSUMPTIONS #84–89.

- **Phase 10:** n8n. `app/integrations/` — `signing` (HMAC both ways, `canonical_json`),
  `notify` (`Notifier`: `review_needed` / `vendor_followup` signed webhooks, retries,
  `notifications` rows), `followup` (vendor-faq templates). `app/api/intake.py`:
  `POST /intake/email` (signed, no API key; idempotent by message id + file name) and
  `GET /integrations`. `RunService(notifier=…)` schedules notices when a run pauses. Tunnel guard
  middleware in `app/main.py`. Migration 0004 applied on Neon. Workflows in `n8n/workflows/`,
  setup in `n8n/README.md`. UI: Settings → Integrations. ASSUMPTIONS #90–98 (#97 = Phase 8 bug
  fixed: older paused cases had no `policy`).
  **Still open:** the user imports the workflows into n8n cloud, connects Gmail, sets `API_KEY`,
  `N8N_WEBHOOK_SECRET`, `N8N_BASE_URL` in `backend/.env` (and `API_KEY` for the frontend), runs
  `cloudflared tunnel --url http://localhost:8000`, fills the Config nodes and activates the
  workflows. Then check the "done when": email a sample PDF → run + reply; reviewer email with the
  deep link.

## Phase 11 — Evaluation

Plan §9/§184: `backend/scripts/eval.py` runs all 25 samples through the graph and reports
field-level extraction accuracy, decision accuracy (vs `ground-truth/*.json` `expected`), average
latency, LLM calls/tokens per invoice, plus the RAG eval (`scripts/eval_rag.py`: hit-rate@5,
answer correctness with an LLM judge). Tune prompts/rules where it fails, document every change
and the results in `docs/EVAL_RESULTS.md`, and show the latest numbers on the dashboard (the
extraction-accuracy KPI is still missing there). Existing pieces: `scripts/eval_extraction.py`
(`--record`/`--replay`, `compare()`), `scripts/run_samples_live.py` (timings), `tests/test_rules_cases.py`.
Reset the demo (or reseed) before a live pass — samples are order-dependent.
**Done when:** `docs/EVAL_RESULTS.md` shows extraction, decision and RAG metrics.
