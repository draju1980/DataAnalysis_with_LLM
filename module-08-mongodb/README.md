# Module 8 — MongoDB (1 week)

[← Module 7](../module-07-semi-structured-text/README.md) · [Syllabus](../README.md) · [Next: Module 9 →](../module-09-combining-modalities-vector-search/README.md)

**You start with:** Modules 1–7 done — MongoDB already holds `llm_calls`, `eval_items`, `eval_results`, `predictions`, `chart_values`, `chart_truth`, `invoices`, `file_errors`, `transcript_segments`, `action_items` and `log_events`.
**You finish with:** `describe_mongo()`, `run_pipeline()` and `ask_mongo()`; Claude answering 10 cross-collection questions with aggregation pipelines it writes itself; and proof that writes are blocked twice — by your checker and by the database user.

## Key ideas (read once)

- **Roles flip.** Until now you checked Claude's output with pipelines. Now Claude writes the pipelines, running as the read-only `course_ro` user.
- **Pipelines vs SQL.** `$match` = WHERE, `$group` = GROUP BY, `$project` = SELECT columns, `$sort`, `$limit`, `$unwind` = expand an array into one document per element, `$lookup` = JOIN.
- **Embed or reference.** Data always read together lives in one document (an invoice's lines). Data shared or growing without limit goes in its own collection and is joined with `$lookup` (eval items and results).
- **Silent empty results.** Claude writes pipelines as JSON text, and plain JSON has no date type: `"2026-09-01"` never matches a stored date. Extended JSON (`{"$date": "2026-09-01T00:00:00Z"}`) parsed with `bson.json_util` fixes this.
- **Guardrails in layers.** The read-only user blocks every write on the server. Your checker also rejects dangerous stages, caps results and sets a timeout, giving Claude clear errors.

> Start of session: `cd DataAnalysis_with_LLM && source .venv/bin/activate && docker compose up -d`

---

## Step 1 — Take inventory of what you've built

Create `m08_inventory.py`:

```python
from claude_multimodal import db_ro

for name in sorted(db_ro.list_collection_names()):
    print(f"{name:22} {db_ro[name].estimated_document_count():>7} docs")
```

```bash
python m08_inventory.py
```

**Check:** you see the collections from Modules 1–7 with document counts above 0 (except any module you skipped).

## Step 2 — Write three pipelines yourself first

Before Claude writes pipelines, write one of each kind so you can judge Claude's. Create `m08_practice.py`:

```python
from claude_multimodal import db_ro

print("1) $match + $group — cost per module:")
for r in db_ro.llm_calls.aggregate([
        {"$group": {"_id": "$module", "cost": {"$sum": "$cost_usd"}, "calls": {"$sum": 1}}},
        {"$sort": {"cost": -1}}]):
    print("  ", r)

print("2) $unwind — most common invoice line descriptions:")
for r in db_ro.invoices.aggregate([
        {"$unwind": "$lines"},                                   # one doc per line item
        {"$group": {"_id": "$lines.desc", "times": {"$sum": 1}, "amount": {"$sum": "$lines.amount"}}},
        {"$sort": {"times": -1}}, {"$limit": 5}]):
    print("  ", r)

print("3) $lookup — action items with the segment count of their recording:")
for r in db_ro.action_items.aggregate([
        {"$lookup": {"from": "transcript_segments", "localField": "recording",
                     "foreignField": "recording", "as": "segs"}},
        {"$project": {"_id": 0, "owner": 1, "item": 1, "segments": {"$size": "$segs"}}},
        {"$limit": 5}]):
    print("  ", r)
```

```bash
python m08_practice.py
```

**Check:** all three print results. Change one stage in each and rerun until you can predict the output.

## Step 3 — Add `describe_mongo()` so Claude knows the data

MongoDB has no fixed schema, so Claude needs a summary: collections, field paths with types (sampled), and what each collection means. Append to `claude_multimodal.py`:

```python
COLLECTION_NOTES = {
    "llm_calls": "One document per Claude API call: module, model, tokens, cost_usd (USD), latency_ms, ts (UTC).",
    "eval_items": "Hand-labeled test texts: _id, text, gold_label (the correct label).",
    "eval_results": "One prediction per eval item per run: run_id, item_id (= eval_items._id), model, prompt_version, predicted, cost_usd.",
    "predictions": "Batch-classified texts: _id (text id), label, model, prompt_version.",
    "chart_values": "Numbers Claude read from chart images: image_file, series, label, value, key.",
    "chart_truth": "True values for some charts, same key as chart_values.",
    "invoices": "One document per PDF: invoice_no, date, vendor, currency, total, lines (array of desc, qty, amount), source_file.",
    "file_errors": "Files that failed processing: source_file, module, error, ts.",
    "transcript_segments": "One segment of a recording transcript: recording, i (order), start_s, end_s (seconds), speaker, text.",
    "action_items": "Action items from recordings: recording, owner, item, at (hh:mm:ss).",
    "log_events": "One log event: ts (UTC), service, severity, message, attrs (object), line, source_file.",
}


def _paths(doc, prefix=""):
    for key, value in doc.items():
        path = f"{prefix}{key}"
        if isinstance(value, dict):
            yield path, "object"
            yield from _paths(value, path + ".")
        elif isinstance(value, list) and value and isinstance(value[0], dict):
            yield path, "array<object>"
            yield from _paths(value[0], path + ".")
        else:
            yield path, type(value).__name__


def describe_mongo(sample=50):
    """Text summary of every collection: count, meaning, field paths and types."""
    parts = []
    for name in sorted(db_ro.list_collection_names()):
        types = {}
        for doc in db_ro[name].aggregate([{"$sample": {"size": sample}}]):
            for path, t in _paths(doc):
                types.setdefault(path, set()).add(t)
        fields = ", ".join(f"{p} ({'/'.join(sorted(t))})" for p, t in sorted(types.items()))
        parts.append(f"## {name} ({db_ro[name].estimated_document_count()} docs)\n"
                     f"{COLLECTION_NOTES.get(name, '')}\nFields: {fields}")
    return "\n\n".join(parts)
```

**Check:** `python -c "from claude_multimodal import describe_mongo; print(describe_mongo())"` prints every collection with its note and fields. Nested fields like `lines.amount` and dates (`datetime`) appear.

## Step 4 — Add `run_pipeline()`, the guarded query runner

Append:

```python
from bson import json_util

BLOCKED_OPERATORS = {"$out", "$merge", "$function", "$accumulator", "$where"}


def _keys(obj):
    if isinstance(obj, dict):
        for key, value in obj.items():
            yield key
            yield from _keys(value)
    elif isinstance(obj, list):
        for value in obj:
            yield from _keys(value)


def run_pipeline(collection, pipeline_json, max_docs=500):
    """Run a read-only aggregation written as Extended JSON text. Returns Extended JSON text."""
    pipeline = json_util.loads(pipeline_json)           # {"$date": ...} -> real datetime
    if not isinstance(pipeline, list):
        raise ValueError("a pipeline must be a JSON array of stages")
    if collection not in db_ro.list_collection_names():
        raise ValueError(f"unknown collection: {collection}")
    blocked = BLOCKED_OPERATORS & set(_keys(pipeline))
    if blocked:
        raise ValueError(f"blocked operators: {sorted(blocked)}")
    pipeline.append({"$limit": max_docs})               # cap the result size
    docs = db_ro[collection].aggregate(pipeline, maxTimeMS=10_000)   # kill slow queries
    return json_util.dumps(list(docs))
```

The four layers: `course_ro` can't write (server), blocked operators (checker), `$limit` (size cap), `maxTimeMS` (timeout).

**Check:** run all four tests:

```bash
python -c "
from claude_multimodal import run_pipeline
print('1 ok:', run_pipeline('llm_calls', '[{\"\$group\": {\"_id\": \"\$module\", \"n\": {\"\$sum\": 1}}}]')[:120])
print('2 date:', run_pipeline('llm_calls', '[{\"\$match\": {\"ts\": {\"\$gte\": {\"\$date\": \"2026-01-01T00:00:00Z\"}}}}, {\"\$count\": \"n\"}]'))
for bad in ['[{\"\$out\": \"copy\"}]',
            '[{\"\$lookup\": {\"from\": \"invoices\", \"pipeline\": [{\"\$merge\": \"x\"}], \"as\": \"y\"}}]']:
    try: run_pipeline('llm_calls', bad)
    except ValueError as e: print('blocked:', e)"
```

Prints a group result, a non-zero date count, and `blocked:` twice (the second hides `$merge` inside `$lookup`).

## Step 5 — Add `ask_mongo()` and ask one question

Append:

```python
PIPELINE_TOOL = {
    "name": "run_pipeline",
    "description": "Run a read-only MongoDB aggregation pipeline on one collection; returns the results as Extended JSON.",
    "input_schema": {"type": "object", "properties": {
        "collection": {"type": "string"},
        "pipeline_json": {"type": "string", "description":
            'JSON array of stages. Write dates as {"$date": "2026-09-01T00:00:00Z"} (UTC).'}},
        "required": ["collection", "pipeline_json"]},
}


def ask_mongo(question, *, model=SONNET):
    """Answer a question by letting Claude write and run pipelines. Returns (answer, pipelines)."""
    system = (DOC_RULE + " You answer questions about a MongoDB database by running aggregation pipelines with "
              "the run_pipeline tool. Every number in your answer must come from a pipeline result in "
              "this conversation. Use $lookup to join collections and $unwind for arrays. Dates are "
              "stored in UTC. If the data can't answer the question, say so.\n\n" + describe_mongo())
    history = [{"role": "user", "content": question}]
    resp, replies = run_with_tools(history, [PIPELINE_TOOL], {"run_pipeline": run_pipeline},
                                   system=system, model=model, module="m8")
    pipelines = [b.input for r in replies for b in r.content if b.type == "tool_use"]
    return text_of(resp), pipelines
```

**Check:**

```bash
python -c "
from claude_multimodal import ask_mongo
answer, pipes = ask_mongo('How much did each module cost in Claude API calls so far?')
print(answer)"
```

Prints `[tool] run_pipeline …` and per-module costs that match `python m08_practice.py` part 1.

## Step 6 — Write 10 questions and answer them yourself

Create `data/m8/questions.txt` (run `mkdir -p data/m8` first) with 10 questions. At least **six must need `$lookup`** (two collections) and **two must need `$unwind`** (an array). Ideas that fit your data:

```
For each evaluation run, what was the accuracy and total cost?
Which eval items were misclassified by every run?
For charts that have truth values, which chart had the largest average error?
Which recordings have action items, and how many per owner?
Which files failed in Module 4 and do any of them also appear in invoices?
Which vendors have invoices whose lines don't add up to the total?
What are the five most common invoice line descriptions by total amount?
How many ERROR events per service per hour?
```

Answer each one **yourself** with your own pipeline (like Step 2) and write the results in `notes/m08_answers.md`.

**Check:** 10 questions, 10 answers you computed yourself.

## Step 7 — Let Claude answer all 10

Create `m08_ask.py`:

```python
import json
from pathlib import Path
from claude_multimodal import ask_mongo

FENCE = "`" * 3                       # a Markdown code fence
out = ["# Module 8 — Claude's answers", ""]
for n, q in enumerate(Path("data/m8/questions.txt").read_text().splitlines(), 1):
    if not q.strip():
        continue
    answer, pipelines = ask_mongo(q)
    out += [f"## {n}. {q}", "", answer, "", "Pipelines used:", ""]
    out += [f"{FENCE}json\n{json.dumps(p, indent=2)}\n{FENCE}" for p in pipelines] + [""]
    print(f"{n}. done")
Path("notes/m08_results.md").write_text("\n".join(out))
print("wrote notes/m08_results.md")
```

```bash
python m08_ask.py
```

**Check:** `notes/m08_results.md` has 10 answers, each with the pipelines that produced it.

## Step 8 — Compare and fix

Compare `notes/m08_answers.md` with `notes/m08_results.md`. For each mismatch, read Claude's pipeline. Common causes: a date written as a plain string, a field name guessed wrong, a `$lookup` on the wrong key. Fix the **input** — usually a clearer line in `COLLECTION_NOTES` — then rerun Step 7.

**Check:** all 10 answers match yours.

## Step 9 — Guardrail test 1: the checker rejects a write request

```bash
python -c "
from claude_multimodal import ask_mongo
answer, pipes = ask_mongo('Copy all ERROR log events into a new collection called error_archive.')
print(answer)"
```

Claude may try `$out` or `$merge`; the checker returns `blocked operators` to it, and it should report that it can't write.

**Check:** `python m08_inventory.py` shows **no** `error_archive` collection.

## Step 10 — Guardrail test 2: the database blocks it even without the checker

Bypass your checker and send `$out` straight to the server as `course_ro`:

```bash
python -c "
from pymongo.errors import OperationFailure
from claude_multimodal import db_ro
try: list(db_ro.log_events.aggregate([{'\$match': {'severity': 'ERROR'}}, {'\$out': 'error_archive'}]))
except OperationFailure as e: print('server blocked it:', e.details.get('errmsg'))"
```

**Check:** prints `server blocked it: not authorized …`. Two independent layers each stopped the write.

## Step 11 — Optional: explore with the MongoDB MCP server

MongoDB's official MCP server lets Claude Desktop or Claude Code browse your database without code. Configure it with your **read-only** connection string (`MONGODB_URI` from `.env`) and its read-only mode enabled — never the `course_rw` string. Follow the setup in MongoDB's MCP server documentation.

**Check:** in Claude Desktop, ask "list the collections in the course database" and get the Step 1 list.

## Step 12 — Commit

```bash
git add claude_multimodal.py m08_*.py notes/m08_answers.md notes/m08_results.md
git commit -m "Module 8: describe_mongo, guarded run_pipeline, ask_mongo"
git push
```

**Check:** pushed.

## Done when

- [ ] Step 8: all 10 answers verified against your own pipelines.
- [ ] Step 9: a write request through `ask_mongo()` is rejected by the checker.
- [ ] Step 10: the same write sent directly is rejected by the server.

**Next:** [Module 9 — Combining modalities and vector search](../module-09-combining-modalities-vector-search/README.md)
