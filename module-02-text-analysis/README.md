# Module 2 — Text analysis (1 week)

[← Module 1](../module-01-claude-api-fundamentals/README.md) · [Syllabus](../README.md) · [Next: Module 3 →](../module-03-images/README.md)

**You start with:** Module 1 done — `ask()`, `tool_input()`, `cost_of()`, `log_call()` in `claude_multimodal.py`.
**You finish with:** `classify()`; a hand-labeled test set in MongoDB; an evaluation script you can rerun after any prompt change; a comparison of two prompt versions; and 500 texts classified with the Batches API at half price.

## Key ideas (read once)

- **Structured output with a forced tool.** Instead of asking for JSON in words, define a tool whose input schema *is* the shape you want and force Claude to call it. An `enum` limits the answer to your fixed labels.
- **Evaluation is the core skill.** Never judge a prompt by eye. Label examples by hand, run the prompt over them, compute accuracy, change one thing, rerun, compare.
- **XML tags separate data from instructions.** Put the text inside `<review>…</review>` so Claude never mistakes it for instructions.

> Start of session: `cd DataAnalysis_with_LLM && source .venv/bin/activate && docker compose up -d`

---

## Step 1 — Install pandas and get 600 short texts

```bash
pip install pandas
mkdir -p data/m2
```

Save 600 product reviews (or news headlines) as `data/m2/reviews.csv` with two columns, `id` and `text`. Public review datasets on Kaggle or Hugging Face work well; keep `id` to letters, digits, `-` or `_`.

```
id,text
r001,"Battery died after two days, very disappointed."
r002,"Does what it says. Nothing special."
...
```

**Check:** `python -c "import pandas as pd; d=pd.read_csv('data/m2/reviews.csv'); print(len(d), list(d.columns))"` prints `600 ['id', 'text']`.

## Step 2 — Choose your labels in one place

Create `m02_config.py`. Every Module 2 script imports from it, so labels never get out of sync.

```python
# Labels Claude may choose from. For news headlines use topics instead,
# e.g. ["business", "technology", "politics", "sports", "other"].
LABELS = ["positive", "negative", "neutral"]
```

**Check:** `python -c "from m02_config import LABELS; print(LABELS)"` prints your labels.

## Step 3 — Label 100 texts by hand

Create a template with the first 100 texts and an empty `label` column:

```bash
python -c "
import pandas as pd
d = pd.read_csv('data/m2/reviews.csv').head(100)
d['label'] = ''
d.to_csv('data/m2/labels.csv', index=False)"
```

Open `data/m2/labels.csv` in Excel, Numbers or LibreOffice, type one label from Step 2 in every row, and save it as CSV. Decide hard cases consistently (e.g. "mixed" reviews → `neutral`). This is your answer key.

**Check:**

```bash
python -c "
import pandas as pd
from m02_config import LABELS
d = pd.read_csv('data/m2/labels.csv')
print(d.label.value_counts()); print('invalid:', (~d.label.isin(LABELS)).sum())"
```

Shows counts per label and `invalid: 0`.

## Step 4 — Load the answer key into MongoDB

Create `m02_load_items.py`:

```python
import pandas as pd
from claude_multimodal import db_rw

labels = pd.read_csv("data/m2/labels.csv")
for row in labels.itertuples():
    db_rw.eval_items.replace_one(
        {"_id": str(row.id)},
        {"_id": str(row.id), "text": row.text, "gold_label": row.label},
        upsert=True)
print("eval_items:", db_rw.eval_items.count_documents({}))
```

```bash
python m02_load_items.py
```

`replace_one(..., upsert=True)` inserts or overwrites, so rerunning after you fix a label is safe.

**Check:** prints `eval_items: 100`.

## Step 5 — Add `classify()` to `claude_multimodal.py`

Append:

```python
PROMPTS = {
    "v1": ("Classify the review inside <review> tags as one of: {labels}.\n"
           "<review>\n{text}\n</review>"),
}


def label_tool(labels):
    """A tool whose only input is one label from a fixed list."""
    return {
        "name": "record_label",
        "description": "Record the single best label for the text.",
        "input_schema": {
            "type": "object",
            "properties": {"label": {"type": "string", "enum": labels}},
            "required": ["label"],
        },
    }


def classify(text, labels, *, model=HAIKU, prompt_version="v1", module="m2"):
    """Return (label, reply). The forced tool call guarantees a valid label."""
    tool = label_tool(labels)
    prompt = PROMPTS[prompt_version].format(labels=", ".join(labels), text=text)
    resp = ask(prompt, model=model, max_tokens=1024, tools=[tool],
               tool_choice={"type": "tool", "name": tool["name"]}, module=module)
    return tool_input(resp)["label"], resp
```

`tool_choice` forces the tool, and the `enum` means the answer can only be one of your labels.

**Check:**

```bash
python -c "
from claude_multimodal import classify
from m02_config import LABELS
print(classify('Arrived broken and support never replied.', LABELS)[0])"
```

Prints `negative` (or the matching label for your set).

## Step 6 — Write the evaluation script

Create `m02_eval.py`. It runs one prompt version over all 100 labeled items and stores every prediction.

```python
import sys
from datetime import datetime, timezone
from claude_multimodal import classify, cost_of, db_rw, HAIKU
from m02_config import LABELS

prompt_version = sys.argv[1]                                 # e.g. v1
model, model_name = HAIKU, "haiku"
run_id = f"{prompt_version}-{model_name}-{datetime.now(timezone.utc):%Y%m%dT%H%M%S}"

db_rw.eval_results.create_index([("run_id", 1), ("item_id", 1)], unique=True)
for n, item in enumerate(db_rw.eval_items.find(), 1):
    label, resp = classify(item["text"], LABELS, model=model, prompt_version=prompt_version)
    db_rw.eval_results.insert_one({
        "run_id": run_id, "item_id": item["_id"], "prompt_version": prompt_version,
        "model": model_name, "predicted": label, "cost_usd": cost_of(resp)})
    print(f"\r{n}/100", end="", flush=True)
print("\nfinished run", run_id)
```

Run it with prompt v1:

```bash
python m02_eval.py v1
```

**Check:** prints `100/100` and `finished run v1-haiku-…`.

## Step 7 — Write the report script

Create `m02_report.py`. One aggregation pipeline joins each prediction to its correct label (`$lookup`), then computes accuracy and cost per run.

```python
import sys
from claude_multimodal import db_ro

join = [
    {"$lookup": {"from": "eval_items", "localField": "item_id",
                 "foreignField": "_id", "as": "item"}},
    {"$unwind": "$item"},                       # one-element array -> plain field
]
report = join + [
    {"$group": {"_id": "$run_id",
                "model": {"$first": "$model"}, "prompt": {"$first": "$prompt_version"},
                "accuracy": {"$avg": {"$cond": [{"$eq": ["$predicted", "$item.gold_label"]}, 1, 0]}},
                "cost_usd": {"$sum": "$cost_usd"}}},
    {"$sort": {"accuracy": -1}},
]
print(f"{'run':40} {'model':7} {'prompt':6} {'accuracy':>8} {'cost $':>9}")
for r in db_ro.eval_results.aggregate(report):
    print(f"{r['_id']:40} {r['model']:7} {r['prompt']:6} {r['accuracy']:8.1%} {r['cost_usd'] or 0:9.5f}")

if len(sys.argv) > 1:                            # python m02_report.py <run_id>
    mistakes = join + [
        {"$match": {"run_id": sys.argv[1], "$expr": {"$ne": ["$predicted", "$item.gold_label"]}}},
        {"$project": {"_id": 0, "predicted": 1, "gold": "$item.gold_label", "text": "$item.text"}},
    ]
    print("\nMistakes:")
    for m in db_ro.eval_results.aggregate(mistakes):
        print(f"- predicted {m['predicted']}, correct {m['gold']}: {m['text'][:120]}")
```

```bash
python m02_report.py
python m02_report.py <run_id from Step 6>
```

**Check:** the first command shows one row with an accuracy and cost; the second also lists the mistakes. Read every mistake — some will be your labeling errors (fix them in `labels.csv` and rerun Step 4).

## Step 8 — Improve the prompt and measure the change

Add a second prompt version with clear rules and examples. In `claude_multimodal.py`, add a `"v2"` entry inside `PROMPTS`:

```python
    "v2": ("Classify the sentiment of the review inside <review> tags as one of: {labels}.\n"
           "Rules: mixed or lukewarm reviews are neutral; judge the product, not the delivery.\n"
           "Examples:\n"
           "<review>Love it, works perfectly.</review> -> positive\n"
           "<review>Stopped working after a week.</review> -> negative\n"
           "<review>Okay for the price, nothing special.</review> -> neutral\n"
           "<review>\n{text}\n</review>"),
```

Adjust the rules and examples to the mistakes you saw in Step 7. Then rerun:

```bash
python m02_eval.py v2
python m02_report.py
```

**Check:** a `v2` row appears. Whether accuracy went up or down, you now *know* — that's the point.

## Step 9 — Classify all 600 with the Batches API

Pick the best prompt version from the report. Batches run in the background (usually minutes, up to 24 hours) at half price. Create `m02_batch.py`:

```python
import sys, time
import pandas as pd
from claude_multimodal import client, db_rw, label_tool, log_call, tool_input, PROMPTS, HAIKU
from m02_config import LABELS

model = HAIKU
prompt_version = sys.argv[1]
tool = label_tool(LABELS)

df = pd.read_csv("data/m2/reviews.csv")
requests = [{
    "custom_id": str(r.id),
    "params": {
        "model": model, "max_tokens": 1024, "tools": [tool],
        "tool_choice": {"type": "tool", "name": tool["name"]},
        "messages": [{"role": "user", "content":
                      PROMPTS[prompt_version].format(labels=", ".join(LABELS), text=r.text)}],
    }} for r in df.itertuples()]

batch = client.messages.batches.create(requests=requests)
print("batch", batch.id, "submitted")
while client.messages.batches.retrieve(batch.id).processing_status != "ended":
    time.sleep(30)
    print("waiting…")

for res in client.messages.batches.results(batch.id):
    if res.result.type != "succeeded":
        print("failed:", res.custom_id, res.result.type)
        continue
    msg = res.result.message
    log_call(msg, "m2-batch", 0, batch=True)
    db_rw.predictions.replace_one(
        {"_id": res.custom_id},
        {"_id": res.custom_id, "label": tool_input(msg)["label"],
         "model": "haiku", "prompt_version": prompt_version}, upsert=True)
print("predictions:", db_rw.predictions.count_documents({}))
```

```bash
python m02_batch.py v2      # use your best prompt version
```

**Check:** prints `predictions: 600`.

## Step 10 — Compare normal vs batch cost per call

```bash
python -c "
from claude_multimodal import db_ro
for r in db_ro.llm_calls.aggregate([
    {'\$match': {'module': {'\$in': ['m2', 'm2-batch']}}},
    {'\$group': {'_id': {'module': '\$module', 'model': '\$model'},
                 'calls': {'\$sum': 1}, 'cost': {'\$sum': '\$cost_usd'}}}]):
    print(r['_id'], r['calls'], 'calls, \$%.5f per call' % (r['cost'] / r['calls']))"
```

**Check:** the `m2-batch` cost per call is about half the `m2` cost per call.

## Step 11 — Write your conclusion and commit

Create `notes/m02_report.md` with three lines: accuracy per prompt version (from Step 8's report), cost for 600 texts each way (Step 10), and which prompt you'd use and why.

```bash
mkdir -p notes
# write notes/m02_report.md
git add m02_*.py claude_multimodal.py notes/m02_report.md
git commit -m "Module 2: classify, evaluation, batch run"
git push
```

**Check:** the commit is pushed; `data/` is not in it.

## Done when

- [ ] Step 7: `python m02_report.py` ranks every run by accuracy and cost from one query.
- [ ] Step 8: you changed the prompt and measured the effect instead of judging by eye.
- [ ] Step 9: all 600 texts classified via the Batches API.
- [ ] Step 11: your choice of prompt is written down with numbers.

**Next:** [Module 3 — Images](../module-03-images/README.md)
