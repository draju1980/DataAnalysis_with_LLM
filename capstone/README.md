# Capstone (about 2 weeks)

[← Module 10](../module-10-production-and-security/README.md) · [Syllabus](../README.md)

**You start with:** all modules done — every function in `lib_claude_multimodal.py`, the evaluation habit, the container and CI setup, and the threat model template.
**You finish with:** one real project that uses at least four input formats, stores results in MongoDB, meets the Module 10 standard, and is written up with accuracy, cost per run and a threat model.

## Before you start or resume

The capstone takes about two weeks. Run these three blocks at the start of **every** session.

**1. Start the session**

```bash
cd DataAnalysis_with_LLM
source .venv/bin/activate
docker compose up -d
until docker compose ps mongodb | grep -q "(healthy)"; do sleep 3; done; echo "MongoDB ready"
```

**2. Check the prerequisites** (all modules)

```bash
python -c "
print('\n')
from lib_claude_multimodal import (ask, extract_pdf_fields, load_tables, ask_image, transcribe, extract_log_records,
    ask_text_file, run_pipeline, build_chunks, create_chunk_index, search_chunks, redact)
print('all module functions ok')" && test -f m09_assistant.py && test -f threat-model.md && echo "M9 assistant and M10 threat model: ok"
```

**Check:** prints both `ok` lines. An `ImportError` names the missing function and so the module to finish.

**3. Find where you stopped**

```bash
(
  step() { if eval "$2" >/dev/null 2>&1; then echo "done  $1"; else echo "todo  $1"; fi; }
  step "Step 1   capstone/PLAN.md"              'test -f capstone/PLAN.md'
  step "Step 2   inputs in data/capstone"       'test -n "$(find data/capstone -type f | head -1)"'
  step "Step 3   capstone/test_questions.csv"   'test -f capstone/test_questions.csv'
  step "Step 4   capstone_ingest.py"            'test -f capstone_ingest.py'
  step "Step 6   capstone_assistant.py"         'test -f capstone_assistant.py'
  step "Step 7   capstone/results.csv"          'test -f capstone_eval.py && test -f capstone/results.csv'
  step "Step 10  threat model + write-up"       'test -f capstone/threat-model.md && test -f capstone/WRITEUP.md'
  step "Step 10  committed"                     'git log --oneline --author="$(git config user.email)" | grep -q "Capstone:"'
)
python -c "
print('\n')
from lib_claude_multimodal import db_ro
r = list(db_ro.llm_calls.aggregate([{'\$match': {'module': 'capstone'}},
    {'\$group': {'_id': None, 'calls': {'\$sum': 1}, 'cost': {'\$sum': '\$cost_usd'}}}]))
print('capstone calls so far:', r[0]['calls'] if r else 0, '| cost \$%.4f' % ((r[0]['cost'] or 0) if r else 0))
for m in db_ro.chunks.aggregate([{'\$group': {'_id': '\$modality', 'n': {'\$sum': 1}}}]): print('chunks:', m)"
```

**Check:** resume at the first `todo` line. Steps 5, 8 and 9 leave no single file: Step 5 is done when the chunk counts include your capstone sources, Step 8 when you have a cost-per-run number, Step 9 when the container, CI and injection checks pass.

**Resuming safely**

- **Make `capstone_ingest.py` safe to rerun**, because you will rerun it: replace or upsert by source file, as Modules 3–5 and 7 do, so a second run never duplicates documents.
- Step 5 calls `build_chunks()`, which deletes every chunk and embedding first. Rerun it only after ingesting new material.
- Step 8 divides by the number of runs. If you ingested more than once while building, count every run, or note the cost before your final clean run and subtract it.
- **To stop for the day**, stop MongoDB (or leave it running; your data stays):

  ```bash
  docker compose stop
  ```

  Never `docker compose down -v`: it deletes the database.

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

**Check:**

```bash
python capstone_ingest.py
python m08_inventory.py
```

The ingest finishes, and the inventory shows your new data.

## Step 5 — Build the search layer

Rebuild chunks and the index from Module 9 so the new material is searchable:

```bash
python -c "
print('\n')
from lib_claude_multimodal import build_chunks, create_chunk_index, SEARCH_ROUTE
print(build_chunks(), 'chunks')
if SEARCH_ROUTE == 'vector':
    from lib_claude_multimodal import embed_chunks; print(embed_chunks(), 'embedded')
print(create_chunk_index(), 'ready')"
```

If your capstone has new sources (e.g. HTML pages), add them to `build_chunks()` first, with a source field that `cite()` can show.

**Check:** the chunk count by modality includes your capstone sources.

## Step 6 — Build the answer script

Copy `m09_assistant.py` to `capstone_assistant.py`. Change the system prompt to describe your project, and add any tool you need (for example `run_sql` from Module 6 for the tables). Keep the rule: every number comes from a tool result, every claim has a source.

**Check:**

```bash
python capstone_assistant.py
```

The assistant answers one of your Step 1 questions with a citation.

## Step 7 — Run the test set and score it

Create `capstone_eval.py` that reads `capstone/test_questions.csv`, asks each question, and writes `capstone/results.csv` with columns `question, expected_answer, system_answer, correct`. Fill the `correct` column (yes/no) by comparing each answer with your answer key and opening its citation.

**Check:** accuracy = correct ÷ 20, written at the top of `capstone/results.csv` or in your notes.

## Step 8 — Measure cost per run

```bash
python -c "
print('\n')
from lib_claude_multimodal import db_ro
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
