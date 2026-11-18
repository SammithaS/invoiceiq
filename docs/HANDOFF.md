# Handoff — where the build stands

Read this first in a new session, then `docs/BUILD_PLAN.md` (the full plan) and
`docs/ASSUMPTIONS.md` (every decision so far, #1–42).

## Status

- Repo `C:\Users\asus\Desktop\invoiceiq`, branch `main`. Phases 1–4 committed.
- **Next: Phase 5 — RAG.**
- Working agreement: finish a phase fully; ruff + mypy + pytest pass; commit per phase on `main`;
  log ambiguities in `docs/ASSUMPTIONS.md`; ask before deploying, paid services or stack changes.
  The user puts API keys in `backend/.env` themselves.

## Environment (Windows)

- Backend commands run from `backend/` as `python -m uv run ...` (`uv` is not on PATH), with
  `UV_PROJECT_ENVIRONMENT=C:/Users/asus/.venvs/invoiceiq` and `PYTHONIOENCODING=utf-8`.
- Checks: `python -m uv run ruff check .`, `ruff format --check .`, `mypy app scripts tests`,
  `pytest -q` (416 passing at the end of Phase 4).
- Neon DB in `backend/.env` (`DATABASE_URL`); `GOOGLE_API_KEY` and `GROQ_API_KEY` are set.
- Dev servers may already be running (backend :8000, frontend :3000) — check ports first.

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

## Phase 5 — RAG

Ingest `data/knowledge/` into PGVector (heading-aware chunks; metadata `source`, `section`,
`vendor_id`, `doc_type`, validity), hybrid retrieval (vector + full-text, RRF), and a
`retrieve_policy` step whose LLM output is `app.rules.models.ContractTerms` (rates with validity
and citations, `rate_revision_citation`, `no_po_citation`). It should reproduce what
`tests/contract_fixture.py` gives today, so `test_rules_cases.py` can run on recorded RAG output.
Deferred contract citations (ASSUMPTIONS #40) come from here. Plus the RAG eval set (hit-rate@5).
**Done when:** edge case #3 flags the price with a correct contract citation.
