# Handoff — where the build stands

Read this first in a new session, then `docs/BUILD_PLAN.md` (the full plan) and
`docs/ASSUMPTIONS.md` (every decision so far, #1–89).

## Status

- Repo `C:\Users\asus\Desktop\invoiceiq`, branch `main`. Phases 1–9 committed.
- **Next: Phase 10 — n8n automations.**
- Working agreement: finish a phase fully; ruff + mypy + pytest pass; commit per phase on `main`;
  log ambiguities in `docs/ASSUMPTIONS.md`; ask before deploying, paid services or stack changes.
  The user puts API keys in `backend/.env` themselves.

## Environment (Windows)

- Backend commands run from `backend/` as `python -m uv run ...` (`uv` is not on PATH), with
  `UV_PROJECT_ENVIRONMENT=C:/Users/asus/.venvs/invoiceiq` and `PYTHONIOENCODING=utf-8`.
- Checks: `python -m uv run ruff check .`, `ruff format --check .`, `mypy app scripts tests`,
  `pytest -q` (506 passing at the end of Phase 9).
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

## Phase 10 — n8n automations

Plan §12: four workflows exported to `n8n/workflows/` — (1) email intake (Gmail/IMAP trigger with a
PDF → HMAC-signed POST to a new `POST /intake/email` → reply "received, run #…"), (2) review
notification (FastAPI calls an n8n webhook when a run reaches WAITING_REVIEW → email/Slack with a
deep link to the run; reminder after 24 h), (3) vendor follow-up on REQUEST_INFO / missing fields
(template from `vendor-faq.md`), (4) daily AP summary from `GET /stats`. Webhooks both ways are
**HMAC-SHA256 signed with a timestamp check** (`N8N_WEBHOOK_SECRET`, `N8N_BASE_URL` already in
`app/config.py`). Runs get `source="EMAIL"`. **Ask the user first** how to run n8n (Docker locally vs
n8n cloud) and which mail account / Slack to use — that's a stack/service decision.
**Done when:** emailing a PDF creates a run, and the reviewer gets a notification with a deep link.
