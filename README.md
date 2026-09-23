# iSERV Agentic Test-Generation Framework

Automatically generates and runs Playwright end-to-end tests for iSERV forms —
driven from a written specification (PRD) plus the application's own source
code and live database, rather than a human hand-writing every test.

This repo now contains two pipelines, the result of merging the `main` and
`new-approach` branches:

| | Current pipeline (repo root) | `legacy_ticket_flows/` |
|---|---|---|
| Approach | PRD-driven, two-stage LLM generation, DB-grounded values, deterministic sanitize backstop | Single-pass LLM generation from source code only |
| Flows implemented | `create_service_report` | `customer_create_ticket`, `agent_create_ticket`, `post_creation_visibility` |
| LLM | OpenAI (gpt-4o for skeletons/SQL plan, gpt-4.1 for populate/analyze) | OpenAI (gpt-4o) |
| Status | Active — this is where new flows should be built | Reference only — kept because it's the only implementation of its three flows |

**If you're starting new work, use the root pipeline.** Porting the three
legacy flows to the new methodology means writing a PRD per flow (see below) —
a deliberate, human-authored input, not a mechanical migration.

Full design rationale, the file-by-file logic of every script, known
limitations, and the detailed roadmap live in `KT_fin.docx` (repository root's
parent folder) — read that before making non-trivial changes to the root
pipeline.

---

## Current Pipeline — How It Works

```
Source Code + Live DB              PRD                  Human (once per flow)
        │                           │                            │
        ▼                           ▼                            │
   extract.py ──────────────────────┼──── context_{flow}.json     │
        │                           │                             │
        │                    generate1.py ──── scenario skeletons │
        │                           │            (the "what")     │
        └──────────────┬────────────┘                             │
                        ▼                                          │
                 generate2.py ──── populated matrix (the "how")    │
                        │                                          │
                        ▼                                          │
                  sanitize.py ──── deterministic repair            │
                        │                                          │
                        ▼                                          ▼
                  record.py  ◄──────────────────────────  human records the form once
                        │
                        ▼
                  execute.py ──── raw_results_{flow}.json
                        │
                        ▼
                   report.py ──── report_{flow}.xlsx
```

The PRD is the source of truth: `generate1.py` reads *only* the PRD to decide
which tests should exist (never the code, so a code bug can't silently become
the "expected" behaviour). `generate2.py` then reads skeletons + code + DB to
fill in real, submittable values. See `KT_fin.docx` §4 for why this split
matters.

### Running it

```powershell
# 1. template -> manifest
python make_manifest.py path/to/template.txt

# 2. extract source slices + live DB data
python extract.py --flow create_service_report

# 3. skeletons (what to test) from the PRD
python generate1.py --flow create_service_report --prd path/to/prd.md

# 4. populate the matrix (real values + recordings plan)
python generate2.py --flow create_service_report

# 5. deterministic repair (no LLM, free to re-run)
python sanitize.py --flow create_service_report

# 6. record once per flow_type, then build the drivers
python record.py --record-all create_service_report
python record.py --build-all  create_service_report

# 7. run + report
python execute.py --flow create_service_report [--dry-run] [--case caseNNN]
python report.py  --flow create_service_report
```

### Setup

```powershell
pip install -r requirements.txt
playwright install chromium
copy .env.example .env      # fill in DB creds, OpenAI key, one test login per role
```

### Output layout

| Folder / file | Written by | Contents |
|---|---|---|
| `manifests/{subdir}/{flow}.json` | `make_manifest` | Which source files to read + read_mode |
| `query_plans/{flow}.json` | `extract` | Cached LLM-derived SQL plan |
| `context/context_{flow}.json` | `extract` | Code slices + DB entities + field labels |
| `skeletons/scenario_skeletons_{flow}.json` | `generate1` | The "what" tree + coverage summary |
| `matrices/scenario_matrix_{flow}.json` | `generate2`/`sanitize` | The full, repaired test matrix |
| `base_tests/base_test_{flow}_{flow_type}[_recorded].py` | `record` | Raw recording + reusable driver |
| `results/raw_results_{flow}.json` | `execute` | Pass/fail/skip per case |
| `report_{flow}.xlsx` | `report` | Final Excel workbook |

---

## `legacy_ticket_flows/` — the earlier pipeline

Self-contained: its own `extract.py`/`generate.py`/`record.py`/`execute.py`/
`report.py`, run from inside that directory. See
`legacy_ticket_flows/README.md` for full usage — it covers the three ticket
flows (`customer_create_ticket`, `agent_create_ticket`,
`post_creation_visibility`) that the current pipeline doesn't have PRDs for
yet. It shares this repo's root `requirements.txt` and `.env` (see comments
in `.env.example` for which variables belong to which pipeline).

---

## Known limitations

See `KT_fin.docx` §8–10 for the full, honest list. Headlines:

- Only proven on one flow (`create_service_report`) across three entry
  points — not yet "any web app with just a manifest."
- Multi-session test cases (`session_count > 1`) are skipped, not automated.
- Ticket/entity selection logic is iSERV-shaped, not yet a generic
  "pick a record satisfying rule X" engine.
- Requires a live app + live DB + real seeded test accounts — no fixture/mock
  mode exists.

## Security notes

- `.env` is gitignored — never commit real credentials.
- `storage_states/` (cached browser session cookies/JWTs) was previously
  tracked in git on both branches; it has been removed from tracking as part
  of this merge and added to `.gitignore`. If this repo has ever been public,
  treat those test accounts' credentials as exposed and rotate them.
