# Handoff — where the build stands

Read this first in a new session, then `docs/BUILD_PLAN.md` (the full plan) and
`docs/ASSUMPTIONS.md` (every decision so far, #1–30).

## Status

- Repo `C:\Users\asus\Desktop\invoiceiq`, branch `main`. Phases 1–3 committed (Phase 3 = `b1f5b76`).
- **Next: Phase 4 — Rules engine.**
- Working agreement: finish a phase fully; ruff + mypy + pytest pass; commit per phase on `main`;
  log ambiguities in `docs/ASSUMPTIONS.md`; ask before deploying, paid services or stack changes.
  The user puts API keys in `backend/.env` themselves.

## Environment (Windows)

- Backend commands run from `backend/` as `python -m uv run ...` (`uv` is not on PATH), with
  `UV_PROJECT_ENVIRONMENT=C:/Users/asus/.venvs/invoiceiq` and `PYTHONIOENCODING=utf-8`.
- Checks: `python -m uv run ruff check .`, `ruff format --check .`, `mypy app scripts tests`,
  `pytest -q` (287 passing at the end of Phase 3).
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

## Phase 4 — rules engine

Pure Python in `backend/app/rules/` (math, gst, vendor/PO matching, tolerance, duplicates, decide),
full unit tests, no LLM. Rule results are PASS/WARN/FAIL with policy citations.
**Done when:** each of the 25 test invoices yields its expected decision and exception categories,
using extracted data from the recorded fixtures plus seed data and earlier cases per
`depends_on` / manifest order.

Policy numbers (`ap-policy.md`; ASSUMPTIONS #14, #15, #17, #18):
- Price tolerance 2% per line and ₹5,000 per invoice; a cited contract rate beats the PO rate.
- Quantity tolerance 0 vs GRN; freight ≤ ₹1,000; rounding ₹1.
- Duplicates: 30-day window; also compare service periods (CloudNest bills monthly).
- Approval matrix: ≤ ₹5 L none; ≤ ₹25 L Finance Manager; above, + CFO. Non-PO: Department Head
  (+ Finance Manager above ₹1 L). Bank change: AP Manager + Finance Controller.
- Precedence: critical FAIL → `REJECTED` (blocked vendor, exact duplicate, math off > ₹1,
  quotation/credit note); missing dependency → `ON_HOLD` (no GRN, bank change, unknown vendor);
  any WARN, key confidence < 0.8, or an approval gate → `NEEDS_REVIEW`; else `APPROVED`.
- `expected.decision` is the decision before any human acts.
- Contract-rate checks (TechNova ₹1,100 vs ₹1,250) need RAG (Phase 5): take contract terms as an
  input and use a fixture until the retriever exists.
- `malabar-erp-mismatch` passes all rules; its ERP fault is injected in Phase 8.
