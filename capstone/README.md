# Capstone (about 2 weeks)

[← Module 10](../module-10-production-and-security/README.md) · [Syllabus](../README.md)

**You start with:** all modules done — every function in `claude_multimodal.py`, the evaluation habit, the container and CI setup, and the threat model template.
**You finish with:** one real project that uses at least four input formats, stores results in MongoDB, meets the Module 10 standard, and is written up with accuracy, cost per run and a threat model.

> Start of session: `cd DataAnalysis_with_LLM && source .venv/bin/activate && docker compose up -d`

---

## Step 1 — Choose a project

| Option | Inputs (at least four formats) | Output |
| --- | --- | --- |
| Daily market brief | Broker contract notes (PDF), trade history (CSV), chart screenshots, earnings call audio | A morning summary with positions, key moves and cited sources |
| Incident analyzer | Logs, Kubernetes events (YAML/JSON), dashboard screenshots, postmortem docs | Timeline, probable root cause and similar past incidents |
| Due-diligence assistant | Company filings (PDF), financials (Excel), investor call recordings, news pages (HTML) | Question answering with a citation for every claim |

Create `capstone/PLAN.md` with: the option, the exact inputs you'll use, and the 3–5 questions the finished system must answer.

**Check:** `capstone/PLAN.md` names four input formats.

## Step 2 — Collect the inputs

```bash
mkdir -p data/capstone/{pdf,tables,images,audio,text}
```

Copy your inputs into the matching folders. Keep private material in `data/` (git-ignored).

**Check:** each folder you need has files in it.

## Step 3 — Write the test set before building anything

Create `capstone/test_questions.csv` with 20 questions you know the answers to:

```
question,expected_answer,source
What was the largest position at close on 30 Sep?,RELIANCE 1200 shares,trades.csv
When did the CFO mention margin pressure?,around 00:23:10,q3-call recording
```

This is your answer key; write it before you see what the system says.

**Check:** 20 rows, each with an expected answer and its source.

## Step 4 — Ingest every format with the functions you already built

Create `capstone_ingest.py` that loads each input with the matching module's function and tags every call with `module="capstone"` where the function accepts it:

| Format | Function (module) | Lands in |
| --- | --- | --- |
| PDF | `extract_pdf_fields()` / folder loop (M4) | `invoices` or a new collection |
| CSV / Excel / JSON | `load_tables()` (M6), copied to MongoDB if needed | DuckDB / new collection |
| Images | `ask_image()` with a schema (M3) | new collection |
| Audio / video | `m05_process.py` (M5) | `transcript_segments`, `action_items` |
| Logs / YAML / HTML | `extract_log_records()` (M7) or `ask_text_file()` | `log_events` |

Record failures in `file_errors` exactly as in Module 4.

**Check:** `python capstone_ingest.py` finishes, and `python m08_inventory.py` shows your new data.

## Step 5 — Build the search layer

Rebuild chunks and the index from Module 9 so the new material is searchable:

```bash
python -c "
from claude_multimodal import build_chunks, create_chunk_index, SEARCH_ROUTE
print(build_chunks(), 'chunks')
if SEARCH_ROUTE == 'vector':
    from claude_multimodal import embed_chunks; print(embed_chunks(), 'embedded')
print(create_chunk_index(), 'ready')"
```

If your capstone has new sources (e.g. HTML pages), add them to `build_chunks()` first, with a source field that `cite()` can show.

**Check:** the chunk count by modality includes your capstone sources.

## Step 6 — Build the answer script

Copy `m09_assistant.py` to `capstone_assistant.py`. Change the system prompt to describe your project, and add any tool you need (for example `run_sql` from Module 6 for the tables). Keep the rule: every number comes from a tool result, every claim has a source.

**Check:** `python capstone_assistant.py` answers one of your Step 1 questions with a citation.

## Step 7 — Run the test set and score it

Create `capstone_eval.py` that reads `capstone/test_questions.csv`, asks each question, and writes `capstone/results.csv` with columns `question, expected_answer, system_answer, correct`. Fill the `correct` column (yes/no) by comparing each answer with your answer key and opening its citation.

**Check:** accuracy = correct ÷ 20, written at the top of `capstone/results.csv` or in your notes.

## Step 8 — Measure cost per run

```bash
python -c "
from claude_multimodal import db_ro
r = next(db_ro.llm_calls.aggregate([{'\$match': {'module': 'capstone'}},
    {'\$group': {'_id': None, 'calls': {'\$sum': 1}, 'cost': {'\$sum': '\$cost_usd'}}}]))
print(r['calls'], 'calls, total \$%.4f' % r['cost'])"
```

Divide by the number of runs (ingest once + 20 questions) to get cost per run.

**Check:** you have one cost-per-run number.

## Step 9 — Meet the Module 10 standard

- Containerize the ingest job like Module 10 Steps 2–4.
- Add a CI gate on a small, publishable slice of your test set like Module 10 Steps 7–10.
- Run one injection test on your own input type like Module 10 Step 12.

**Check:** the container job runs, CI is green, and the injection test has no effect.

## Step 10 — Write the threat model and the write-up

Copy `threat-model.md` to `capstone/threat-model.md` and adapt it to your project. Then create `capstone/WRITEUP.md`:

```markdown
# Capstone — <project name>

## Problem and inputs
## How it works (one diagram or list of steps)
## Accuracy
<n>/20 correct on the test set; what went wrong in the misses.
## Cost per run
<$ amount>, from llm_calls; the biggest cost driver and how to cut it.
## Security
Summary of capstone/threat-model.md and which tests passed.
## What I'd do next
```

```bash
git add capstone/ capstone_*.py
git commit -m "Capstone: <project name>"
git push
```

**Check:** `capstone/WRITEUP.md` has accuracy, cost per run and a link to the threat model.

## Done when

- [ ] Step 7: accuracy on 20 hand-checked questions.
- [ ] Step 8: cost per run from `llm_calls`.
- [ ] Step 9: container, CI gate and injection test.
- [ ] Step 10: threat model and write-up pushed.
