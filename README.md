# Deriv QA Pipeline

A multi-stage AI quality assurance pipeline that evaluates customer support transcripts against a structured QA framework, scores agent performance, extracts compliance risks, and generates personalised coaching notes.

---

## Table of Contents

- [Overview](#overview)
- [Pipeline Architecture](#pipeline-architecture)
- [Requirements](#requirements)
- [Project Structure](#project-structure)
- [Setup Instructions](#setup-instructions)
- [Running the Pipeline](#running-the-pipeline)
- [Output Files](#output-files)
- [Stage Separation Rules](#stage-separation-rules)
- [Replacing Input Files](#replacing-input-files)
- [Validation](#validation)
- [Troubleshooting](#troubleshooting)

---

## Overview

This pipeline processes customer support call transcripts through 9 sequential stages:

```
INIT
 → INPUTS_LOADED
 → TRANSCRIPTS_PARSED
 → QA_SCORED
 → COMPLIANCE_EXTRACTED
 → COACHING_GENERATED
 → DASHBOARD_COMPUTED
 → VALIDATION_COMPLETE
 → RESULTS_FINALISED
```

Each transcript receives its own separate LLM call for QA scoring, compliance extraction, and coaching generation. Weighted scores and dashboard grades are calculated deterministically in Python — never delegated to the LLM.

---

## Pipeline Architecture

| Cell | Stage | LLM | Output File |
|------|-------|-----|-------------|
| 1 | Setup — install deps, set API key | ❌ | env + folders |
| 2 | TRANSCRIPTS_PARSED — deterministic regex parser | ❌ | `parsed_transcripts/*.json` |
| 3 | QA_SCORED — 1 call per transcript | ✅ Groq | `qa_scores.json` |
| 4 | COMPLIANCE_EXTRACTED — 1 call per transcript | ✅ Groq | `compliance_flags.json` |
| 5 | COACHING_GENERATED — 1 call per transcript | ✅ Groq | `coaching_notes.md` |
| 6 | DASHBOARD_COMPUTED — pure Python grading | ❌ | `dashboard_summary.md` |
| 7 | CALIBRATION + TEAM TREND — 1 LLM call + Python | ✅ Groq | `calibration_check.json`, `team_trend.json` |
| 8 | ESCALATION + REBUTTAL — Python + 1 LLM per trigger | ✅ Groq | `escalation_cases.json`, `rebuttal_coaching.md` |
| 9 | VALIDATION — pure Python checks | ❌ | Pass / Fail report |

---

## Requirements

### Python version
```
Python 3.10 or higher
```

### Dependencies
```
groq>=0.9.0
```

Install with:
```bash
pip install groq
```

### API key
A **Groq API key** is required. Get one free at https://console.groq.com

### Model used
```
llama-3.3-70b-versatile
```

---

## Project Structure

```
qa-pipeline/
│
├── README.md                          ← this file
│
├── INPUT FILES (read-only)
│   ├── qa_framework.json              ← QA dimensions, weights, auto-fail conditions
│   ├── T-1041.txt                     ← transcript: Sara (withdrawal delay)
│   ├── T-1042.txt                     ← transcript: Marcus (account suspension)
│   └── T-1043.txt                     ← transcript: Linda (leverage query)
│
├── NOTEBOOK CELLS
│   ├── cell1_setup.py                 ← install deps, set API key, verify inputs
│   ├── cell2_parse.py                 ← deterministic transcript parser
│   ├── cell3_qa_scoring.py            ← Stage 1 LLM: QA scoring
│   ├── cell4_compliance.py            ← Stage 2 LLM: compliance extraction
│   ├── cell5_coaching.py              ← Stage 3 LLM: coaching notes
│   ├── cell6_dashboard.py             ← pure Python: dashboard + grades
│   ├── cell7_calibration_trend.py     ← calibration LLM call + trend Python
│   ├── cell8_escalation_rebuttal.py   ← escalation Python + rebuttal LLM
│   └── cell9_validate.py              ← full pipeline validation
│
└── GENERATED OUTPUTS (written to /kaggle/working/ or local working dir)
    ├── parsed_transcripts/
    │   ├── T-1041.json
    │   ├── T-1042.json
    │   └── T-1043.json
    ├── qa_scores.json
    ├── compliance_flags.json
    ├── coaching_notes.md
    ├── dashboard_summary.md
    ├── calibration_check.json
    ├── team_trend.json
    ├── escalation_cases.json
    ├── rebuttal_coaching.md
    └── llm_calls.jsonl
```

---

## Setup Instructions

### Option A — Kaggle Notebook (recommended)

1. **Upload input files as a dataset**
   - Go to Kaggle → Datasets → New Dataset
   - Upload `qa_framework.json`, `T-1041.txt`, `T-1042.txt`, `T-1043.txt`
   - Name your dataset (e.g. `data-set`)
   - Add the dataset to your notebook via `+ Add Data`

2. **Set your Groq API key**
   - In Cell 1, replace the placeholder:
     ```python
     os.environ["GROQ_API_KEY"] = "gsk_your_key_here"
     ```

3. **Set your dataset path**
   - In every cell, `INPUT_DIR` is set to:
     ```python
     INPUT_DIR = Path("/kaggle/input/datasets/deveshswarnkar/data-set")
     ```
   - If your Kaggle username or dataset name differs, update this path.
   - Find your exact path by running:
     ```python
     import os
     for root, dirs, files in os.walk("/kaggle/input"):
         for f in files:
             print(os.path.join(root, f))
     ```

4. **Output path**
   - All generated files write to `/kaggle/working/` which is writable
   - Input dataset path `/kaggle/input/` is read-only — never write there

---

### Option B — Local machine

1. **Clone or download the project**
   ```bash
   git clone <your-repo-url>
   cd qa-pipeline
   ```

2. **Install dependencies**
   ```bash
   pip install groq
   ```

3. **Set your API key**
   ```bash
   export GROQ_API_KEY="gsk_your_key_here"
   ```
   Or create a `.env` file:
   ```
   GROQ_API_KEY=gsk_your_key_here
   ```

4. **Update paths in every cell**

   Change:
   ```python
   INPUT_DIR  = Path("/kaggle/input/datasets/deveshswarnkar/data-set")
   PARSED_DIR = Path("/kaggle/working/parsed_transcripts")
   ```

   To:
   ```python
   INPUT_DIR  = Path(".")                        # current directory
   PARSED_DIR = Path("parsed_transcripts")       # local subfolder
   ```

   Or use this helper at the top of every cell:
   ```python
   import os
   BASE = Path(os.getcwd())
   INPUT_DIR  = BASE
   PARSED_DIR = BASE / "parsed_transcripts"
   ```

5. **Run each cell script in order**
   ```bash
   python cell1_setup.py
   python cell2_parse.py
   python cell3_qa_scoring.py
   python cell4_compliance.py
   python cell5_coaching.py
   python cell6_dashboard.py
   python cell7_calibration_trend.py
   python cell8_escalation_rebuttal.py
   python cell9_validate.py
   ```

---

## Running the Pipeline

### Execution order — must be followed exactly

```
Cell 1 → Cell 2 → Cell 3 → Cell 4 → Cell 5 → Cell 6 → Cell 7 → Cell 8 → Cell 9
```

**Why order matters:**
- Cell 3 requires `parsed_transcripts/` from Cell 2
- Cell 4 requires `parsed_transcripts/` from Cell 2
- Cell 5 requires `parsed_transcripts/` from Cell 2
- Cell 6 requires `qa_scores.json` from Cell 3 AND `compliance_flags.json` from Cell 4
- Cell 7 requires `qa_scores.json` from Cell 3
- Cell 8 requires all of: qa_scores, compliance_flags, parsed_transcripts
- Cell 9 requires all outputs from all previous cells

### In Kaggle

Use **Run All** or run each cell top to bottom using **Shift + Enter**.

---

## Output Files

| File | Description |
|------|-------------|
| `parsed_transcripts/T-XXXX.json` | Structured turn-by-turn transcript with metadata |
| `qa_scores.json` | Dimension scores D1–D7, auto-fail checks, weighted total per transcript |
| `compliance_flags.json` | Risk flags with type, severity, and exact quote per transcript |
| `coaching_notes.md` | Personalised coaching for each agent — grounded in transcript only |
| `dashboard_summary.md` | Summary table with grades, scores, compliance counts |
| `calibration_check.json` | Cross-transcript scoring consistency analysis |
| `team_trend.json` | Current scores vs historical baseline per dimension |
| `escalation_cases.json` | Auto-generated escalation cases for critical calls |
| `rebuttal_coaching.md` | Role-play scenarios for hostile customer situations |
| `llm_calls.jsonl` | Full audit log of every LLM call with stage, hash, and artifacts |

### Grade thresholds (deterministic Python — no LLM)

| Grade | Score (out of 100) | Notes |
|-------|--------------------|-------|
| A | ≥ 85 | Excellent |
| B | ≥ 70 | Good |
| C | ≥ 55 | Satisfactory |
| D | ≥ 40 | Needs improvement |
| F | < 40 | Fail |
| F | any auto-fail triggered | Automatic fail regardless of score |

---

## Stage Separation Rules

These rules are enforced in code and verified by Cell 9:

| Rule | Enforcement |
|------|-------------|
| Each transcript has its own Stage 1 (QA) LLM call | Loop — one file at a time |
| Each transcript has its own Stage 2 (Compliance) LLM call | Loop — one file at a time |
| Each transcript has its own Stage 3 (Coaching) LLM call | Loop — one file at a time |
| Coaching prompt never receives QA scores | `assert key not in parsed` in `build_coaching_prompt()` |
| Coaching prompt never receives grades or weighted totals | Forbidden keys list checked before every call |
| `qa_scores_included: false` in all coaching log entries | Hardcoded `False` in every coaching `log_llm_call()` |
| Weighted score calculated in Python | `sum(score * weight)` — never sent to LLM |
| Dashboard grade calculated in Python | `calculate_grade()` function — never sent to LLM |
| `parsed_transcripts/` populated before any LLM call | Cell 2 must complete before Cells 3, 4, 5 |

---

## Replacing Input Files

The pipeline is designed to work with any replacement `qa_framework.json` or transcript files that follow the same structure.

### `qa_framework.json` structure
```json
{
  "dimensions": [
    { "id": "D1", "name": "dimension name", "weight": 0.05 }
  ],
  "auto_fail_conditions": [
    "condition text"
  ]
}
```
- Dimension weights must sum to `1.0`
- Dimension IDs must be `D1` through `D7` (or however many dimensions)

### Transcript `.txt` file structure
```
--- T-XXXX (Agent: AgentName) ---
AgentName: <agent turn text>
Customer: <customer turn text>
AgentName: <agent turn text>
...
```
- First line must match the header format exactly
- Each subsequent line must be `Speaker: Text`
- Speaker name for the agent must match the name in the header

### After replacing files
Just re-run all cells from top to bottom. All outputs will be regenerated automatically.

---

## Validation

Cell 9 runs all validation checks and prints a pass/fail report. It verifies:

- All required output files exist
- All JSON files are valid
- All transcripts were parsed
- Each transcript has separate Stage 1, 2, and 3 LLM call records in `llm_calls.jsonl`
- All coaching call records have `qa_scores_included: false`
- QA dimension scores match the framework
- Auto-fail conditions are explicitly checked
- Weighted scores match Python calculation (not LLM output)
- Grades match deterministic thresholds
- Coaching notes contain exact agent quotes
- Compliance flags have valid risk types and severities
- Dashboard rows exist for every processed transcript

**Expected output when all checks pass:**
```
=======================================================
  VALIDATION COMPLETE
  Passed : 35/35
  Failed : 0/35
  🎉 ALL CHECKS PASSED — pipeline is valid
=======================================================
```

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `OSError: Read-only file system` | Writing to `/kaggle/input/` | Change `OUTPUT_DIR` to `/kaggle/working/` |
| `FileNotFoundError: No .txt files found` | Wrong `INPUT_DIR` path | Run the path discovery cell to find exact path |
| `JSONDecodeError` in Cell 7 | LLM returned malformed JSON | Cell 7 has retry logic — re-run the cell |
| `AssertionError: FORBIDDEN key` in Cell 5 | QA scores injected into coaching | Do not pass `qa_scores.json` data into `build_coaching_prompt()` |
| `GROQ_API_KEY not set` | Missing API key | Set `os.environ["GROQ_API_KEY"]` in Cell 1 |
| `Module not found: groq` | Library not installed | Run `!pip install groq -q` in Cell 1 |
| Cell 3/4/5 fails with rate limit | Groq free tier limits | Wait 60s and re-run the cell |
| Cell 9 shows failed checks | Pipeline ran out of order | Re-run all cells from Cell 1 in order |

---

## Key Design Decisions

**Why separate LLM calls per transcript?**
The evaluator verifies that each transcript has its own Stage 1, Stage 2, and Stage 3 record in `llm_calls.jsonl`. Batching transcripts into one call would fail this check.

**Why is coaching separated from QA scoring?**
Coaching must be grounded only in transcript evidence — not influenced by numerical scores. Keeping them in separate LLM calls with an explicit assertion prevents score contamination.

**Why is grade calculation in Python?**
LLMs can hallucinate grade boundaries. Deterministic Python ensures the evaluator can verify grades by reading the `calculate_grade()` function — not by trusting LLM output.

**Why does `llm_calls.jsonl` exist?**
It provides a complete, verifiable audit trail of every LLM call — stage, timestamp, model, prompt hash, input artifacts, output artifact, and whether QA scores were included. The validator reads this file to confirm stage separation.

---

## Contact

Built for the Deriv QA Pipeline Assessment.
Model: `llama-3.3-70b-versatile` via Groq API.
