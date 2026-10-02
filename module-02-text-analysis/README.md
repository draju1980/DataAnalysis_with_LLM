# Module 2 — Text analysis (1 week)

[← Module 1](../module-01-claude-api-fundamentals/README.md) · [Syllabus](../README.md) · [Next: Module 3 →](../module-03-images/README.md)

**You start with:** Module 1 done — `ask()`, `tool_input()`, `cost_of()`, `log_call()` in `claude_multimodal.py`.
**You finish with:** `classify()`; a hand-labeled test set in MongoDB; an evaluation script you can rerun after any prompt change; a comparison of two prompt versions; and 500 texts classified with the Batches API at half price.

## Key ideas (read once)

- **Structured output with a forced tool.** Instead of asking for JSON in words, define a tool whose input schema *is* the shape you want and force Claude to call it. An `enum` limits the answer to your fixed labels.
- **Evaluation is the core skill.** Never judge a prompt by eye. Label examples by hand, run the prompt over them, compute accuracy, change one thing, rerun, compare.
- **XML tags separate data from instructions.** Put the text inside `<review>…</review>` so Claude never mistakes it for instructions.

## Before you start or resume

This module takes about a week, so you'll stop and restart several times. Run these three blocks at the start of **every** session.

**1. Start the session**

```bash
cd DataAnalysis_with_LLM
source .venv/bin/activate
docker compose up -d
until docker compose ps mongodb | grep -q "(healthy)"; do sleep 3; done; echo "MongoDB ready"
```

**2. Check the prerequisites** (Module 1)

```bash
python -c "
from claude_multimodal import ask, text_of, cost_of, log_call, tool_input, db_ro
print('Module 1 ok;', db_ro.llm_calls.count_documents({}), 'calls logged so far')"
```

**Check:** prints `Module 1 ok`. An `ImportError` names the Module 1 function that's missing.

**3. Find where you stopped**

```bash
(
  step() { if eval "$2" >/dev/null 2>&1; then echo "done  $1"; else echo "todo  $1"; fi; }
  step "Step 1   data/m2/reviews.csv + pandas"  'test -f data/m2/reviews.csv && python -c "import pandas"'
  step "Step 2   m02_config.py"                 'test -f m02_config.py'
  step "Step 3   data/m2/labels.csv"            'test -f data/m2/labels.csv'
  step "Step 4   m02_load_items.py"             'test -f m02_load_items.py'
  step "Step 5   classify()"                    'grep -qF "def classify(" claude_multimodal.py'
  step "Step 6   m02_eval.py"                   'test -f m02_eval.py'
  step "Step 7   m02_report.py"                 'test -f m02_report.py'
  step "Step 8   prompt v2"                     'grep -qF "\"v2\":" claude_multimodal.py'
  step "Step 9   m02_batch.py"                  'test -f m02_batch.py'
  step "Step 11  notes/m02_report.md"           'test -f notes/m02_report.md'
  step "Step 11  committed"                     'git log --oneline --author="$(git config user.email)" | grep -q "Module 2:"'
)
python -c "
from claude_multimodal import db_ro
print('eval_items: ', db_ro.eval_items.count_documents({}), '(Step 4 wants 100)')
for r in db_ro.eval_results.aggregate([{'\$group': {'_id': '\$run_id', 'n': {'\$sum': 1}}}, {'\$sort': {'_id': 1}}]):
    print('eval run:   ', r['_id'], r['n'], 'items (Steps 6 and 8 want 100 each)')
print('predictions:', db_ro.predictions.count_documents({}), '(Step 9 wants 600)')"
```

**Check:** resume at the first `todo` line, or at the first count that's short.

**Resuming safely**

- **Step 3 (hand labeling) can take several sessions.** Save `labels.csv` often. Don't rerun the command that creates the template once you've started: it overwrites the file and erases your labels.
- Step 4 is safe to rerun: `replace_one(..., upsert=True)` overwrites, never duplicates.
- Each `m02_eval.py` run gets a new `run_id`. If a run stopped before `100/100`, the counts above show it with fewer than 100 items, and it would skew the report. Delete it, then rerun:

  ```bash
  python -c "
  from claude_multimodal import db_rw
  print(db_rw.eval_results.delete_many({'run_id': '<the short run id>'}).deleted_count, 'deleted')"
  ```

- **Step 9 runs on Anthropic's side.** If your terminal closes while the script prints `waiting…`, the batch keeps running and you still pay for it. Don't submit a new one: copy the `batch msgbatch_…` id the script printed and resume with `python m02_batch.py v2 <batch id>`. (If it stopped while storing results, the rerun logs those calls to `llm_calls` a second time. `predictions` stays correct, and Step 10's per-call average is unaffected.) Lost the id? List your recent batches:

  ```bash
  python -c "
  from claude_multimodal import client
  for b in client.messages.batches.list(limit=5): print(b.id, b.processing_status, b.created_at)"
  ```

- Not sure your `claude_multimodal.py` is right after a break? Compare it with the [complete file for this module](#complete-claude_multimodalpy-after-module-2) at the end of the page.
- **To stop for the day**, run `docker compose stop` or leave MongoDB running. Never `docker compose down -v`: it deletes the database.

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

if len(sys.argv) > 2:                      # resume: python m02_batch.py v2 <batch id>
    batch = client.messages.batches.retrieve(sys.argv[2])
else:
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

If the script is interrupted, the batch keeps running: rerun it with the printed id, `python m02_batch.py v2 <batch id>`, to wait for that batch and collect its results instead of paying for a new one.

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

## Complete `claude_multimodal.py` after Module 2

Use this to cross-check your file once the steps are done, or after a break. It is every block the course has told you to add to `claude_multimodal.py` through Module 2, in order, with the earlier edits applied. The `# ── Module N, Step M ──` lines only show which step added the code below them; your file doesn't need them.

Your `"v2"` prompt will differ: Step 8 asks you to adapt its rules and examples to your own mistakes.

To compare automatically, save the file below as `data/expected.py` (`data/` is git-ignored, so it never gets committed), then:

```bash
diff -Bw <(grep -v '^# ── ' data/expected.py) claude_multimodal.py && echo "your file matches"
```

`-Bw` ignores blank lines and spacing. Every other line `diff` prints is a real difference: a missing step, a block pasted twice, or a typo.

<details>
<summary>Show the complete file (191 lines)</summary>

```python
# ── Module 1, Step 1 ──
"""Shared helpers for the Multimodal Data Analysis with Claude course."""
import os
import time
from datetime import datetime, timezone

import anthropic
from dotenv import load_dotenv
from pymongo import MongoClient

load_dotenv()  # reads .env from the project root

HAIKU = "claude-haiku-4-5-20251001"     # the one model used throughout the course

# USD per million tokens: (input, output). Check Anthropic's pricing page.
PRICES = {
    HAIKU: (1.00, 5.00),
}

client = anthropic.Anthropic(max_retries=4, timeout=120.0)
db_rw = MongoClient(os.environ["MONGODB_URI_RW"]).course   # scripts write with this
db_ro = MongoClient(os.environ["MONGODB_URI"]).course      # queries read with this


# ── Module 1, Step 2 ──
def text_of(resp):
    """Join all text blocks of a reply into one string."""
    return "".join(b.text for b in resp.content if b.type == "text")


def _price(model):
    for name, price in PRICES.items():
        if model.startswith(name) or name.startswith(model):
            return price
    return (None, None)


def cost_of(resp, batch=False):
    """Cost of one reply in USD, or None if the model's price isn't filled in."""
    pin, pout = _price(resp.model)
    if pin is None or pout is None:
        print(f"WARNING: no price for {resp.model}; fill in PRICES")
        return None
    cost = (resp.usage.input_tokens * pin + resp.usage.output_tokens * pout) / 1e6
    return cost * 0.5 if batch else cost   # the Batches API costs half


# ── Module 1, Step 3 ──
def log_call(resp, module, latency_ms, batch=False):
    """Save tokens, cost and latency of one reply to the llm_calls collection."""
    db_rw.llm_calls.insert_one({
        "ts": datetime.now(timezone.utc),       # a real datetime, never a string
        "module": module,
        "model": resp.model,
        "input_tokens": resp.usage.input_tokens,
        "output_tokens": resp.usage.output_tokens,
        "cost_usd": cost_of(resp, batch),
        "latency_ms": latency_ms,
        "stop_reason": resp.stop_reason,
    })


# ── Module 1, Step 4 ──
def ask(prompt=None, *, messages=None, system=None, model=HAIKU, max_tokens=1024,
        temperature=None, tools=None, tool_choice=None, module="adhoc"):
    """Send one request to Claude, log it, and return the reply."""
    if messages is None:
        messages = [{"role": "user", "content": prompt}]
    args = dict(model=model, max_tokens=max_tokens, messages=messages)
    if system:
        args["system"] = system
    if temperature is not None:
        args["temperature"] = temperature
    if tools:
        args["tools"] = tools
    if tool_choice:
        args["tool_choice"] = tool_choice
    start = time.perf_counter()
    resp = client.messages.create(**args)
    log_call(resp, module, int((time.perf_counter() - start) * 1000))
    if resp.stop_reason == "max_tokens":
        print(f"WARNING: reply cut off at max_tokens={max_tokens}")
    return resp


# ── Module 1, Step 8 ──
import ast
import operator

_OPS = {ast.Add: operator.add, ast.Sub: operator.sub, ast.Mult: operator.mul,
        ast.Div: operator.truediv, ast.Mod: operator.mod, ast.Pow: operator.pow,
        ast.USub: operator.neg, ast.UAdd: operator.pos}


def calc(expression):
    """Evaluate +, -, *, /, %, ** on numbers only."""
    def ev(node):
        if isinstance(node, ast.Expression):
            return ev(node.body)
        if isinstance(node, ast.Constant) and isinstance(node.value, (int, float)):
            return node.value
        if isinstance(node, ast.BinOp) and type(node.op) in _OPS:
            left, right = ev(node.left), ev(node.right)
            if isinstance(node.op, ast.Pow) and abs(right) > 100:
                raise ValueError("exponent too large")
            return _OPS[type(node.op)](left, right)
        if isinstance(node, ast.UnaryOp) and type(node.op) in _OPS:
            return _OPS[type(node.op)](ev(node.operand))
        raise ValueError("only numbers and + - * / % ** are allowed")
    result = ev(ast.parse(expression.replace(",", ""), mode="eval"))
    return round(result, 10) if isinstance(result, float) else result   # 92.35, not 92.35000000000001


CALC_TOOL = {
    "name": "calculator",
    "description": "Evaluate an arithmetic expression. Use it for every calculation.",
    "input_schema": {
        "type": "object",
        "properties": {"expression": {"type": "string", "description": "e.g. 1847 * 0.05"}},
        "required": ["expression"],
    },
}


# ── Module 1, Step 9 ──
def tool_input(resp):
    """Return the input of the first tool_use block (used with forced tool calls)."""
    return next(b.input for b in resp.content if b.type == "tool_use")


def run_with_tools(history, tools, handlers, *, module, model=HAIKU, system=None,
                   max_tokens=1024, max_rounds=10):
    """Run Claude with tools until it answers. Returns (final reply, all replies)."""
    replies = []
    for _ in range(max_rounds):
        resp = ask(messages=history, tools=tools, model=model, system=system,
                   max_tokens=max_tokens, module=module)
        replies.append(resp)
        history.append({"role": "assistant", "content": resp.content})
        if resp.stop_reason != "tool_use":
            return resp, replies
        results = []
        for block in resp.content:
            if block.type != "tool_use":
                continue
            print(f"  [tool] {block.name} {block.input}")
            try:
                output = handlers[block.name](**block.input)
                results.append({"type": "tool_result", "tool_use_id": block.id,
                                "content": str(output)})
            except Exception as e:  # send the error back so Claude can fix its call
                results.append({"type": "tool_result", "tool_use_id": block.id,
                                "content": f"Error: {e}", "is_error": True})
        history.append({"role": "user", "content": results})
    raise RuntimeError("too many tool rounds")


# ── Module 2, Step 5 ──
PROMPTS = {
    "v1": ("Classify the review inside <review> tags as one of: {labels}.\n"
           "<review>\n{text}\n</review>"),
    "v2": ("Classify the sentiment of the review inside <review> tags as one of: {labels}.\n"
           "Rules: mixed or lukewarm reviews are neutral; judge the product, not the delivery.\n"
           "Examples:\n"
           "<review>Love it, works perfectly.</review> -> positive\n"
           "<review>Stopped working after a week.</review> -> negative\n"
           "<review>Okay for the price, nothing special.</review> -> neutral\n"
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

</details>


## Done when

- [ ] Step 7: `python m02_report.py` ranks every run by accuracy and cost from one query.
- [ ] Step 8: you changed the prompt and measured the effect instead of judging by eye.
- [ ] Step 9: all 600 texts classified via the Batches API.
- [ ] Step 11: your choice of prompt is written down with numbers.

**Next:** [Module 3 — Images](../module-03-images/README.md)
