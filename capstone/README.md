# Capstone (about 2 weeks)

[← Module 10](../module-10-production-and-security/README.md) · [Syllabus](../README.md)

**You start with:** all modules done — every function in `lib_claude_multimodal.py`, the evaluation habit, the container and CI setup, and the threat model template.
**You finish with:** one real project that uses at least four input formats, stores results in MongoDB, meets the Module 10 standard, and is written up with accuracy, cost per run and a threat model.

**What this lab is about.** The capstone project puts the whole course together on a problem you choose. You feed at least four kinds of input (PDFs, tables, images, audio, logs) through the functions you already built, make them searchable, and answer real questions with a source for every claim. Then you measure it the way Module 10 taught: accuracy on questions you answered yourself first, cost per run from `llm_calls`, a container, a CI gate and an injection test. The write-up is the proof that the system works and what it costs.

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
- Step 5 calls `build_chunks()`, which deletes every chunk and rebuilds them from the sources. Rerun it only after ingesting new material.
- Step 8 divides by the number of runs. If you ingested more than once while building, count every run, or note the cost before your final clean run and subtract it.
- **To stop for the day**, stop MongoDB (or leave it running; your data stays):

  ```bash
  docker compose stop
  ```

  Never `docker compose down -v`: it deletes the database.

---

## Step 1 — Choose a project

**What you're doing:** picking one realistic project and writing down exactly which inputs it uses and which questions it must answer. A short, concrete plan keeps the two weeks focused and tells you when you're done.

| Option | Inputs (at least four formats) | Output |
| --- | --- | --- |
| Daily market brief | Broker contract notes (PDF), trade history (CSV), chart screenshots, earnings call audio | A morning summary with positions, key moves and cited sources |
| Incident analyzer | Logs, Kubernetes events (YAML/JSON), dashboard screenshots, postmortem docs | Timeline, probable root cause and similar past incidents |
| Due-diligence assistant | Company filings (PDF), financials (Excel), investor call recordings, news pages (HTML) | Question answering with a citation for every claim |

Create `capstone/PLAN.md` with: the option, the exact inputs you'll use, and the 3–5 questions the finished system must answer.

Show your plan:

```bash
cat capstone/PLAN.md
```

**Check:** it names the option, at least four input formats, and 3–5 questions.

## Step 2 — Collect the inputs

**What you're doing:** gathering your project's files in one place, one folder per format, inside `data/` so private material never gets committed.

Make the folders:

```bash
mkdir -p data/capstone/{pdf,tables,images,audio,text}
```

Copy your inputs into the matching folders. Keep private material in `data/` (git-ignored).

List what you collected:

```bash
find data/capstone -type f
```

**Check:** each folder you need (at least four) lists at least one file, for example `data/capstone/pdf/contract-note-30sep.pdf` or `data/capstone/audio/q3-call.wav`.

## Step 3 — Write the test set before building anything

**What you're doing:** writing 20 questions with answers you've found yourself, plus where each answer lives. Doing this before you build means the system's output can't sway your answer key, so the accuracy you report in Step 7 is honest.

Create `capstone/test_questions.csv` with 20 questions you know the answers to:

```
question,expected_answer,source
What was the largest position at close on 30 Sep?,RELIANCE 1200 shares,trades.csv
When did the CFO mention margin pressure?,around 00:23:10,q3-call recording
```

This is your answer key; write it before you see what the system says.

Count the lines:

```bash
wc -l capstone/test_questions.csv
```

**Check:** prints `21 capstone/test_questions.csv` (the header plus 20 questions), and every row has an expected answer and a source.

## Step 4 — Ingest every format with the functions you already built

**What you're doing:** writing one script that loads every input into MongoDB using the functions from earlier modules, instead of writing new extraction code. Tagging the calls with `module="capstone"` lets you measure the project's cost in Step 8.

Create `capstone_ingest.py` that loads each input with the matching module's function and tags every call with `module="capstone"` where the function accepts it:

| Format | Function (module) | Lands in |
| --- | --- | --- |
| PDF | `extract_pdf_fields()` / folder loop (M4) | `invoices` or a new collection |
| CSV / Excel / JSON | `load_tables()` (M6), copied to MongoDB if needed | DuckDB / new collection |
| Images | `ask_image()` with a schema (M3) | new collection |
| Audio / video | `m05_process.py` (M5) | `transcript_segments`, `action_items` |
| Logs / YAML / HTML | `extract_log_records()` (M7) or `ask_text_file()` | `log_events` |

Record failures in `file_errors` exactly as in Module 4.

Run the ingest:

```bash
python capstone_ingest.py
```

**Check:** it finishes without a traceback; any file it couldn't read is reported and saved to `file_errors`, not left to crash the run.

List the collections and their sizes (the Module 8 inventory script):

```bash
python m08_inventory.py
```

**Check:** one line per collection with its document count, and the collections your ingest writes to (for example `invoices`, `transcript_segments`, `log_events` or a new one) show more documents than before.

## Step 5 — Build the search layer

**What you're doing:** rebuilding Module 9's `chunks` collection and full-text index so the material you just ingested can be searched and cited.

Rebuild chunks and the index from Module 9 so the new material is searchable:

```bash
python -c "
print('\n')
from lib_claude_multimodal import build_chunks, create_chunk_index
print(build_chunks(), 'chunks')
print(create_chunk_index(), 'ready')"
```

If your capstone has new sources (e.g. HTML pages), add them to `build_chunks()` first, with a source field that `cite()` can show.

**Check:** prints a total such as `1234 chunks`, then `chunks_text ready`. Then count chunks by type:

```bash
python -c "
print('\n')
from lib_claude_multimodal import db_ro
for m in db_ro.chunks.aggregate([{'\$group': {'_id': '\$modality', 'n': {'\$sum': 1}}}]): print(m)"
```

**Check:** one line per modality, such as `{'_id': 'pdf', 'n': 57}`, and the counts are higher than before your ingest (or include the new modality you added).

## Step 6 — Build the answer script

**What you're doing:** adapting the Module 9 assistant to your project, so Claude can search your documents and query your data, and must cite a source for every claim.

Copy the Module 9 assistant:

```bash
cp m09_assistant.py capstone_assistant.py
```

In `capstone_assistant.py`, change the system prompt to describe your project, and add any tool you need (for example `run_sql` from Module 6 for the tables). Keep the rule: every number comes from a tool result, every claim has a source. Also change `module="m9"` to `module="capstone"`, so Step 8 counts these calls. Then start it:

```bash
python capstone_assistant.py
```

Ask one of your Step 1 questions at the `question>` prompt. The script keeps asking until you type `quit` or `exit`.

**Check:** the answer cites a source you can open, such as a file plus page, time or line, or the collection it queried.

## Step 7 — Run the test set and score it

**What you're doing:** asking the system all 20 test questions in one go and comparing each answer with your answer key. This gives the accuracy number for your write-up and shows where the system goes wrong.

Create `capstone_eval.py` that reads `capstone/test_questions.csv`, asks each question, and writes `capstone/results.csv` with columns `question, expected_answer, system_answer, correct`. Fill the `correct` column (yes/no) by comparing each answer with your answer key and opening its citation.

Run it:

```bash
python capstone_eval.py
```

**Check:** `capstone/results.csv` has 20 rows. After you fill the `correct` column, accuracy = correct ÷ 20, written at the top of `capstone/results.csv` or in your notes.

## Step 8 — Measure cost per run

**What you're doing:** adding up what the capstone's Claude calls cost, from the `llm_calls` log, and turning it into a cost per run so you can say what the system costs to operate.

```bash
python -c "
print('\n')
from lib_claude_multimodal import db_ro
r = next(db_ro.llm_calls.aggregate([{'\$match': {'module': 'capstone'}},
    {'\$group': {'_id': None, 'calls': {'\$sum': 1}, 'cost': {'\$sum': '\$cost_usd'}}}]))
print(r['calls'], 'calls, total \$%.4f' % r['cost'])"
```

**Check:** prints one line such as `412 calls, total $0.8731`. A `StopIteration` error means no call was tagged `module="capstone"` yet.

Divide by the number of runs (ingest once + 20 questions) to get cost per run.

**Check:** you have one cost-per-run number.

## Step 9 — Meet the Module 10 standard

**What you're doing:** applying Module 10's production checks to your project: run the ingest in a container, gate prompt changes in CI, and prove one planted instruction in your own input type has no effect.

- Containerize the ingest job like Module 10 Steps 2–4.
- Add a CI gate on a small, publishable slice of your test set like Module 10 Steps 7–10.
- Run one injection test on your own input type like Module 10 Step 12.

**Check:** the container job runs, CI is green, and the injection test has no effect.

## Step 10 — Write the threat model and the write-up

**What you're doing:** recording the risks of your project and the results (accuracy, cost, security) in two short documents, then publishing everything. The write-up is what someone else reads to judge the capstone.

Copy the Module 10 threat model, then adapt it to your project:

```bash
cp threat-model.md capstone/threat-model.md
```

Then create `capstone/WRITEUP.md`:

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

Stage the capstone files:

```bash
git add capstone/ capstone_*.py
```

Commit them; replace `daily market brief` with your project's name, keeping `Capstone:` at the start (the progress check looks for it):

```bash
git commit -m "Capstone: daily market brief"
```

Push to GitHub:

```bash
git push
```

**Check:** `capstone/WRITEUP.md` has accuracy, cost per run and a link to the threat model.

## Done when

- [ ] Step 7: accuracy on 20 hand-checked questions.
- [ ] Step 8: cost per run from `llm_calls`.
- [ ] Step 9: container, CI gate and injection test.
- [ ] Step 10: threat model and write-up pushed.
