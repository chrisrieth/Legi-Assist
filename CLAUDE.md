# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project shape

Two loosely-coupled halves that share the `data/` directory as a contract:

1. **Python pipeline** (`run_pipeline.py`, `pipeline/`) — scrapes Maryland General Assembly (MGA), converts PDFs → Markdown, applies amendments via LLM, runs QA via LLM, and writes `data/{year}rs/frontend_data.json`.
2. **Static Vue frontend** (`index.html`) — single 2,200-line file using Vue 3 + MDWDS + Tailwind loaded from CDNs. Fetches `data/{year}rs/frontend_data.json` at runtime. Deployed as a pre-built static site via Vercel (`vercel.json` has `buildCommand: null`, `outputDirectory: "."`).

## Common commands

```bash
# Pipeline (primary entry point — idempotent, safe to re-run)
python run_pipeline.py --year 2026 --model-family gemini
python run_pipeline.py --year 2026 --debug          # first 10 bills only
python run_pipeline.py --model-family claude --model claude-sonnet-4-6
# model-family choices: gemini | gpt | claude | ollama

# Rebuild agency metadata CSV/JSON (uses Gemini w/ Google Search grounding)
python describe_agencies.py           # fills missing rows
python describe_agencies.py --rerun   # backfill acronyms/aliases

# Frontend CSS (only build step the site has)
npm run build:css        # one-shot minified build
npm run watch:css        # watch mode during development
```

There is no test suite, no linter config, and no frontend bundler — editing `index.html` is live-editing the deployed app once committed.

## Pipeline architecture — the non-obvious parts

**State is the source of truth, not the filesystem.** `pipeline/state.py` persists `data/{year}rs/pipeline_state.json`, keyed by `BillNumber`. Each record carries `needs_download | needs_convert | needs_amend | needs_qa` flags plus content hashes (`bill_hash`, `amend_input_hash`, `qa_input_hash`).

**Cascading dirty propagation.** `state.mark_dirty(bill, stage)` sets the flag for that stage *and all downstream stages*. So when `download.py` sees new PDFs, it calls `mark_dirty(bill, 'convert')` which cascades to amend and qa. Each stage is responsible for clearing only its own `needs_*` flag after running — don't touch downstream flags except through `mark_dirty`.

**Hash gates inside stages.** `amend.py` and `qa.py` additionally compare input hashes and short-circuit if unchanged, even when the flag is set. When modifying those stages, preserve the hash check or you'll burn LLM spend on re-runs.

**Crossfile dedup.** `download.py` sorts bills alphabetically (HB before SB) and drops any bill whose number appears as another bill's `CrossfileBillNumber`. The filtered list is what's written back to `legislation.json` and drives the whole run.

**Strikethrough handling is load-bearing.** `convert.py::pdf_page_to_markdown` detects strikethrough rectangles in PDF drawings and wraps struck words as `~~word~~`. Downstream prompts (`pipeline/qa.py::SYSTEM_PROMPT`) explicitly tell the LLM that `~` means stricken text. The fiscal note uses the simpler `pdf_text_simple` path — no strikethrough detection — because fiscal notes don't use them.

**QA fallback chain.** `run_qa` prefers `{bill}_amended.md`, falls back to `{bill}.md`, and if neither exists synthesizes a mini-doc from `legislation.json` (title/synopsis/subjects). The fiscal note (`{bill}_fn.md`) is always appended when present. This means QA can still produce output for bills whose PDFs failed to download.

**Agency relevance uses a closed enum.** `pipeline/qa.py` reads `data/maryland_agencies.csv` at import time and builds `Literal[tuple(unique_agencies)]` for the `AgencyRelevance.agency_name` field. Editing that CSV changes the schema the LLM must match — keep agency names stable.

**Frontend export is a separate final step.** `run_pipeline.py::export_frontend_data` merges `legislation.json` + per-bill `qa_results` from state into `frontend_data.json`. The frontend reads this file directly; state file is not consumed by the frontend.

## LLM client abstraction

`llm_utils.py::query_llm_with_retries` is the single call site for all LLM families. It takes a Pydantic `response_format` and handles structured-output quirks per provider (Gemini `response_schema`, OpenAI `beta.chat.completions.parse`, Claude `messages.parse`, Ollama `format=json_schema`). When adding a new stage, route through this helper rather than calling SDKs directly.

## Data layout

```
data/
  maryland_agencies.csv / .json   # agency list + LLM-generated summaries
  {year}rs/
    legislation.json              # deduped master list (written by download stage)
    pipeline_state.json           # per-bill state + hashes
    frontend_data.json            # final merged output, consumed by index.html
    pdf/                          # downloaded source PDFs
    md/                           # converted markdown + {bill}_amended.md
```

## Deployment & automation

- `.github/workflows/daily_pipeline.yml` runs the pipeline nightly at 05:00 UTC on the upstream repo only (`if: github.repository == 'Maryland-State-Innovation-Team/Legi-Assist'`), then commits `data/` changes back to `main`. Local pushes to `main` will conflict with its commits — pull before pushing.
- Vercel serves the repo root as-is; any commit to `main` deploys. There is no build step, so changes to `index.html` or `data/**` go live immediately.
