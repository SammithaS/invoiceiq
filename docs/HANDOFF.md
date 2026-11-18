# Handoff — where the build stands

Read this first in a new session, then `docs/BUILD_PLAN.md` (the full plan) and
`docs/ASSUMPTIONS.md` (every decision so far, #1–62).

## Status

- Repo `C:\Users\asus\Desktop\invoiceiq`, branch `main`. Phases 1–6 committed.
- **Next: Phase 7 — Live run view, review queue, dashboard.**
- Working agreement: finish a phase fully; ruff + mypy + pytest pass; commit per phase on `main`;
  log ambiguities in `docs/ASSUMPTIONS.md`; ask before deploying, paid services or stack changes.
  The user puts API keys in `backend/.env` themselves.

## Environment (Windows)

- Backend commands run from `backend/` as `python -m uv run ...` (`uv` is not on PATH), with
  `UV_PROJECT_ENVIRONMENT=C:/Users/asus/.venvs/invoiceiq` and `PYTHONIOENCODING=utf-8`.
- Checks: `python -m uv run ruff check .`, `ruff format --check .`, `mypy app scripts tests`,
  `pytest -q` (475 passing at the end of Phase 6).
- Neon DB in `backend/.env` (`DATABASE_URL`); `GOOGLE_API_KEY` and `GROQ_API_KEY` are set.
- Dev servers may already be running (backend :8000, frontend :3000) — check ports first.
  The backend must run with `--loop app.loop:selector_loop_factory` on Windows (psycopg).

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

## Phase 7 — Live run view, review queue, dashboard

UI pages from plan §13 on the Phase 6 API: `/` upload + "Try a sample" (`GET /samples`,
`POST /samples/{key}/run`), `/runs/[id]` live view (`GET /runs/{id}` for the snapshot, then
`EventSource` on `/api/backend/runs/{id}/events?after=<last seq>`; events: `step_started`,
`step_finished`, `rule`, `paused`, `status`, `ping`), PDF preview from `GET /runs/{id}/file` with
evidence boxes from `state.extraction.evidence`, decision card from `state.assessment` +
`explanation`, inline Approve/Reject (`POST /runs/{id}/review`), `/review` queue
(`GET /runs?status=WAITING_REVIEW`), `/dashboard`. Regenerate the typed client
(`scripts/export_openapi.py` → `frontend/lib/api/schema.d.ts`). Check the Next.js proxy route
streams SSE unbuffered. Speed up `retrieve_policy` (#62) so samples finish < 30 s.
**Done when:** every sample runs live in the UI under ~30 seconds, and review works end-to-end.
