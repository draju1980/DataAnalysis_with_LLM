# Module 3 — Images (4–5 days)

[← Module 2](../module-02-text-analysis/README.md) · [Syllabus](../README.md) · [Next: Module 4 →](../module-04-pdfs-and-documents/README.md)

**You start with:** Module 2 done — `ask()`, `tool_input()` and the evaluation habit.
**You finish with:** `image_block()` and `ask_image()`; numbers extracted from 20 charts into MongoDB; a measured error rate; and a written decision on which image tasks can run without a human check.

## Key ideas (read once)

- **Images go in as content blocks**, base64-encoded, next to your text question. Supported: JPEG, PNG, GIF, WebP.
- **Bigger images cost more tokens.** Resize before sending; there is no benefit above about 1568 px on the long side.
- **Know what to trust.** Vision is strong at description, layout and printed text; weaker at exact counts, tiny text and precise values read off a chart. You measure this in Step 9.

## Before you start or resume

You'll likely spread this module over several sessions. Run these three blocks at the start of **every** session.

**1. Start the session**

```bash
cd DataAnalysis_with_LLM
source .venv/bin/activate
docker compose up -d
until docker compose ps mongodb | grep -q "(healthy)"; do sleep 3; done; echo "MongoDB ready"
```

**2. Check the prerequisites** (Modules 1–2)

```bash
python -c "
print('\n')
from lib_claude_multimodal import ask, text_of, tool_input, classify
print('Modules 1-2 ok')"
```

**Check:** prints `Modules 1-2 ok`. An `ImportError` names the missing function and so the module to finish.

**3. Find where you stopped**

```bash
(
  step() { if eval "$2" >/dev/null 2>&1; then echo "done  $1"; else echo "todo  $1"; fi; }
  step "Step 1   20 charts + Pillow"            'test $(ls data/m3/charts | wc -l) -ge 20 && python -c "import PIL"'
  step "Step 2   image_block()"                 'grep -qF "def image_block(" lib_claude_multimodal.py'
  step "Step 3   ask_image()"                   'grep -qF "def ask_image(" lib_claude_multimodal.py'
  step "Step 4   m03_size_cost.py"              'test -f m03_size_cost.py'
  step "Step 5   CHART_SCHEMA"                  'grep -qF "CHART_SCHEMA =" lib_claude_multimodal.py'
  step "Step 6   m03_extract.py"                'test -f m03_extract.py'
  step "Step 7   data/m3/truth.csv"             'test -f data/m3/truth.csv'
  step "Step 8   m03_load_truth.py"             'test -f m03_load_truth.py'
  step "Step 9   m03_errors.py"                 'test -f m03_errors.py'
  step "Step 11  data/m3/screenshot.png"        'test -f data/m3/screenshot.png'
  step "Step 12  notes/m03_decision.md"         'test -f notes/m03_decision.md'
  step "Step 12  committed"                     'git log --oneline --author="$(git config user.email)" | grep -q "Module 3:"'
)
python -c "
print('\n')
from lib_claude_multimodal import db_ro
print('charts extracted:', len(db_ro.chart_values.distinct('image_file')), '(Step 6 wants 20)')
print('chart_truth rows:', db_ro.chart_truth.count_documents({}), '(Step 8 wants your truth.csv row count)')"
```

**Check:** resume at the first `todo` line, or at the first count that's short.

**Resuming safely**

- **Never rerun `m03_export_truth.py` after you've typed in true values.** It overwrites `data/m3/truth.csv` and erases your corrections. If you need to export again, rename the old file first.
- `m03_extract.py` is safe to rerun: it replaces each chart's values instead of adding to them. It does call the API for all 20 charts again.
- Step 8 is safe to rerun: it empties `chart_truth` before loading.
- Not sure your `lib_claude_multimodal.py` is right after a break? Compare it with the [complete file for this module](#complete-lib_claude_multimodalpy-after-module-3) at the end of the page.
- **To stop for the day**, run `docker compose stop` or leave MongoDB running. Never `docker compose down -v`: it deletes the database.

---

## Step 1 — Install Pillow and collect 20 charts

```bash
pip install pillow
mkdir -p data/m3/charts
```

Put 20 chart screenshots (bar, line, pie; PNG or JPG) in `data/m3/charts/`. Pick charts whose true numbers you can find, e.g. from reports or dashboards you have the data for.

**No charts of your own? Generate the course sample.** This draws 20 bar, line, pie and grouped-bar charts from fixed numbers, and saves those numbers in `data/m3/chart_source.csv`. That file is your source of truth in Step 7. Install matplotlib first, so the script runs and your editor can resolve its imports:

```bash
pip install matplotlib
```

Create `m03_make_charts.py`:

```python
"""Make 20 practice charts in data/m3/charts/ and record their true values in data/m3/chart_source.csv."""
import random
from pathlib import Path

import matplotlib
matplotlib.use("Agg")                     # draw to files, no window
import matplotlib.pyplot as plt
import pandas as pd

random.seed(7)                            # same charts and numbers on every run
out = Path("data/m3/charts")
out.mkdir(parents=True, exist_ok=True)

MONTHS = ["Jan", "Feb", "Mar", "Apr", "May", "Jun"]
REGIONS = ["North", "South", "East", "West"]
PRODUCTS = ["Laptops", "Phones", "Tablets", "Monitors", "Printers"]

rows = []                                 # one row per plotted value: the answer key
for n in range(1, 21):
    name = f"chart{n:02d}.png"
    kind = ["bar", "line", "pie", "grouped"][(n - 1) % 4]
    fig, ax = plt.subplots(figsize=(7, 4.5), dpi=110)
    if kind == "bar":
        labels, series = PRODUCTS, "Units sold"
        values = [random.randint(20, 400) for _ in labels]
        bars = ax.bar(labels, values, color="#4C78A8")
        if n % 8 == 1:                    # some charts print the numbers, some don't
            ax.bar_label(bars)
        ax.set_ylabel(series)
        rows += [(name, series, l, v) for l, v in zip(labels, values)]
    elif kind == "line":
        labels = MONTHS
        for series in ("Revenue (k$)", "Costs (k$)"):
            values = [random.randint(50, 250) for _ in labels]
            ax.plot(labels, values, marker="o", label=series)
            rows += [(name, series, l, v) for l, v in zip(labels, values)]
        ax.legend(); ax.grid(alpha=0.3)
    elif kind == "pie":
        labels, series = REGIONS, "Share of sales (%)"
        cuts = sorted(random.sample(range(5, 95), 3))
        values = [a - b for a, b in zip(cuts + [100], [0] + cuts)]   # four shares that sum to 100
        ax.pie(values, labels=labels, autopct="%d%%")
        rows += [(name, series, l, v) for l, v in zip(labels, values)]
    else:                                 # grouped bars: two series side by side
        labels = REGIONS
        width = 0.38
        for i, series in enumerate(("2025", "2026")):
            values = [random.randint(10, 90) for _ in labels]
            ax.bar([x + (i - 0.5) * width for x in range(len(labels))], values, width, label=series)
            rows += [(name, series, l, v) for l, v in zip(labels, values)]
        ax.set_xticks(range(len(labels)), labels); ax.legend(); ax.set_ylabel("Orders")
    ax.set_title(f"Chart {n}: {kind} chart")
    fig.tight_layout()
    fig.savefig(out / name)
    plt.close(fig)

pd.DataFrame(rows, columns=["image_file", "series", "label", "value"]).to_csv(
    "data/m3/chart_source.csv", index=False)
print(f"wrote 20 charts to {out}/ and {len(rows)} true values to data/m3/chart_source.csv")
```

```bash
python m03_make_charts.py
```

**Check:**

```bash
ls data/m3/charts | wc -l
```

Output prints:

```
20
```

## Step 2 — Add `image_block()` to `lib_claude_multimodal.py`

It resizes an image and wraps it as an API content block. Append:

```python
import base64
import io
from pathlib import Path
from PIL import Image


def image_block(path, max_side=1568):
    """Resize an image so its long side is at most max_side px and return a content block."""
    img = Image.open(path)
    img.thumbnail((max_side, max_side))           # keeps aspect ratio; never enlarges
    fmt = "PNG" if img.mode in ("RGBA", "LA", "P") else "JPEG"
    if fmt == "JPEG" and img.mode != "RGB":
        img = img.convert("RGB")
    buf = io.BytesIO()
    img.save(buf, format=fmt)
    return {"type": "image",
            "source": {"type": "base64", "media_type": f"image/{fmt.lower()}",
                       "data": base64.b64encode(buf.getvalue()).decode()}}
```

**Check:**

```bash
python -c "
print('\n')
from pathlib import Path
from lib_claude_multimodal import image_block
p = sorted(Path('data/m3/charts').iterdir())[0]
b = image_block(p); print(p.name, b['source']['media_type'], len(b['source']['data']), 'chars')"
```

Prints a file name, a media type and a size.

## Step 3 — Add `ask_image()` and describe one chart

Append:

```python
def ask_image(paths, question, *, schema=None, tool_name="record", model=HAIKU,
              max_side=1568, max_tokens=512, module="m3"):
    """Ask about one or more images. With a schema, return structured fields (a dict)."""
    if isinstance(paths, (str, Path)):
        paths = [paths]
    content = [image_block(p, max_side) for p in paths] + [{"type": "text", "text": question}]
    messages = [{"role": "user", "content": content}]
    if schema is None:
        return text_of(ask(messages=messages, model=model, max_tokens=max_tokens, module=module))
    tool = {"name": tool_name, "description": "Record the extracted data.", "input_schema": schema}
    resp = ask(messages=messages, model=model, max_tokens=max_tokens, tools=[tool],
               tool_choice={"type": "tool", "name": tool_name}, module=module)
    return tool_input(resp)
```

**Check:**

```bash
python -c "
print('\n')
from pathlib import Path
from lib_claude_multimodal import ask_image
p = sorted(Path('data/m3/charts').iterdir())[0]
print(ask_image(p, 'Describe this chart: type, title, axes, and the main trend.'))"
```

Prints a sensible description of your first chart.

## Step 4 — See how image size drives cost

Send the same chart at two sizes and compare input tokens. Create `m03_size_cost.py`:

```python
from pathlib import Path
from lib_claude_multimodal import ask, image_block

p = sorted(Path("data/m3/charts").iterdir())[0]
for side in (400, 1568):
    r = ask(messages=[{"role": "user", "content": [image_block(p, side),
            {"type": "text", "text": "What is the chart title?"}]}], max_tokens=512, module="m3")
    print(f"max_side={side}: {r.usage.input_tokens} input tokens -> {r.content[0].text}")
```

```bash
python m03_size_cost.py
```

**Check:** the 1568 px version uses several times more input tokens. Note whether the small version still read the title correctly — small images are cheaper but lose fine text.

## Step 5 — Define the chart schema and extract one chart

Append the schema to `lib_claude_multimodal.py`:

```python
CHART_SCHEMA = {
    "type": "object",
    "properties": {
        "title": {"type": ["string", "null"]},
        "points": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "series": {"type": "string", "description": "legend name, or 'value' if only one series"},
                    "label": {"type": "string", "description": "x-axis label or category, exactly as printed"},
                    "value": {"type": "number"},
                },
                "required": ["series", "label", "value"],
            },
        },
    },
    "required": ["title", "points"],
}

CHART_PROMPT = ("Extract every data point shown in this chart. Use series names and axis labels "
                "exactly as printed. If a value is not printed, estimate it from the axis.")
```

**Check:**

```bash
python -c "
print('\n')
from pathlib import Path
from lib_claude_multimodal import ask_image, CHART_SCHEMA, CHART_PROMPT
p = sorted(Path('data/m3/charts').iterdir())[0]
d = ask_image(p, CHART_PROMPT, schema=CHART_SCHEMA, tool_name='record_chart')
print(d['title']); [print(x) for x in d['points'][:5]]"
```

Prints the title and the first data points as dicts.

## Step 6 — Extract all 20 charts into MongoDB

Create `m03_extract.py`:

```python
from pathlib import Path
from lib_claude_multimodal import ask_image, db_rw, CHART_SCHEMA, CHART_PROMPT

IMAGE_TYPES = {".png", ".jpg", ".jpeg", ".gif", ".webp"}
for path in sorted(Path("data/m3/charts").iterdir()):
    if path.suffix.lower() not in IMAGE_TYPES:
        continue
    try:
        data = ask_image(path, CHART_PROMPT, schema=CHART_SCHEMA, tool_name="record_chart")
    except Exception as e:
        print("FAILED", path.name, e)
        continue
    docs = [{"image_file": path.name, "title": data["title"], "series": p["series"],
             "label": p["label"], "value": p["value"],
             "key": f"{path.name}|{p['series']}|{p['label']}"} for p in data["points"]]
    db_rw.chart_values.delete_many({"image_file": path.name})   # rerun-safe
    if docs:
        db_rw.chart_values.insert_many(docs)
    print(f"{path.name}: {len(docs)} values")
```

```bash
python m03_extract.py
```

The `key` field (file | series | label) is what you join on in Step 9.

**Check:** the script prints 20 lines of `<file>: N values`. Then run:

```bash
python -c "print('\n'); from lib_claude_multimodal import db_ro; print(db_ro.chart_values.count_documents({}))"
```

Output prints the total.

## Step 7 — Export 5 charts' values to fill in the truth

Choose 5 charts whose real numbers you know. Export what Claude read so the series and labels match exactly. Create `m03_export_truth.py`:

```python
import sys
import pandas as pd
from lib_claude_multimodal import db_ro

rows = list(db_ro.chart_values.find({"image_file": {"$in": sys.argv[1:]}},
                                    {"_id": 0, "image_file": 1, "series": 1, "label": 1, "value": 1}))
pd.DataFrame(rows).to_csv("data/m3/truth.csv", index=False)
print(len(rows), "rows written to data/m3/truth.csv")
```

```bash
python m03_export_truth.py chart01.png chart02.png chart03.png chart04.png chart05.png
```

Open `data/m3/truth.csv` and **overwrite the `value` column with the true numbers** from your source data. Leave the other columns unchanged.

If you generated the sample charts in Step 1, the true numbers are in `data/m3/chart_source.csv`: find the row with the same `image_file`, `series` and `label`. Claude may name a series or label slightly differently (e.g. `Revenue` instead of `Revenue (k$)`); match them by meaning, and keep Claude's spelling in `truth.csv`.

**Check:** the CSV has rows for your 5 charts and you have corrected every value.

## Step 8 — Load the truth into MongoDB

Create `m03_load_truth.py`:

```python
import pandas as pd
from lib_claude_multimodal import db_rw

t = pd.read_csv("data/m3/truth.csv")
t["key"] = t.image_file + "|" + t.series.astype(str) + "|" + t.label.astype(str)
db_rw.chart_truth.delete_many({})
db_rw.chart_truth.insert_many(t.to_dict("records"))
print("chart_truth:", db_rw.chart_truth.count_documents({}))
```

```bash
python m03_load_truth.py
```

**Check:** prints the same row count as Step 7.

## Step 9 — Measure the error rate

Create `m03_errors.py`:

```python
from lib_claude_multimodal import db_ro

rows = list(db_ro.chart_values.aggregate([
    {"$lookup": {"from": "chart_truth", "localField": "key", "foreignField": "key", "as": "truth"}},
    {"$unwind": "$truth"},
    {"$project": {"_id": 0, "key": 1, "read": "$value", "truth": "$truth.value",
                  "rel_error": {"$cond": [{"$eq": ["$truth.value", 0]}, None,
                      {"$divide": [{"$abs": {"$subtract": ["$value", "$truth.value"]}},
                                   {"$abs": "$truth.value"}]}]}}},
    {"$sort": {"rel_error": -1}},
]))
errs = [r["rel_error"] for r in rows if r["rel_error"] is not None]
print(f"{len(rows)} values checked")
print(f"exact (<0.5% off): {sum(e < 0.005 for e in errs)}  |  >5% off: {sum(e > 0.05 for e in errs)}")
print("\nWorst 10:")
for r in rows[:10]:
    print(f"  {r['key']}: read {r['read']}, true {r['truth']}, off {r['rel_error']:.1%}")
```

```bash
python m03_errors.py
```

**Check:** prints how many values were exact, how many were more than 5% off, and the 10 worst misses. Look at those charts: are they dense, small-text, unlabeled bars, log scales?

## Step 10 — Compare two images in one request

`ask_image()` takes a list, so Claude can compare images directly:

```bash
python -c "
print('\n')
from pathlib import Path
from lib_claude_multimodal import ask_image
a, b = sorted(Path('data/m3/charts').iterdir())[:2]
print(ask_image([a, b], 'The first image is chart A, the second chart B. What changed between them?'))"
```

**Check:** the answer refers to both charts by A and B.

## Step 11 — Read text from a screenshot (OCR)

Take a screenshot with printed text (a terminal, an error dialog, a receipt) and save it as `data/m3/screenshot.png`.

**No screenshot handy?** Download Tesseract's standard OCR test image, a scanned paragraph in English, French, Italian, German, Spanish and Dutch:

```bash
curl -fL -o data/m3/screenshot.png https://tesseract-ocr.github.io/tessdoc/images/eurotext.png
```

```bash
python -c "
print('\n')
from lib_claude_multimodal import ask_image
print(ask_image('data/m3/screenshot.png', 'Transcribe all text in this image exactly, preserving line breaks.'))"
```

**Check:** compare the output to the screenshot line by line; note any misread characters.

## Step 12 — Decide what can run unattended, and commit

Create `notes/m03_decision.md` answering, with your Step 9 numbers: which image tasks you'd run without a human check (e.g. titles, printed text, labeled values) and which need review (e.g. values estimated from bar heights).

```bash
git add lib_claude_multimodal.py m03_*.py notes/m03_decision.md
git commit -m "Module 3: image extraction and error measurement"
git push
```

**Check:** pushed; no images from `data/` in the commit.

## Complete `lib_claude_multimodal.py` after Module 3

Use this to cross-check your file once the steps are done, or after a break. It is every block the course has told you to add to `lib_claude_multimodal.py` through Module 3, in order, with the earlier edits applied. The `# ── Module N, Step M ──` lines show which step added the code below them. Each step's block starts with its own marker line, so pasting it keeps your file labelled in step order; if your file is missing some markers, that's fine — the diff below ignores them.

To compare automatically, save the file below as `data/expected.py` (`data/` is git-ignored, so it never gets committed), then:

```bash
diff -Bw <(grep -v '^# ── ' data/expected.py) <(grep -v '^# ── ' lib_claude_multimodal.py) && echo "your file matches"
```

`-Bw` ignores blank lines and spacing. Every other line `diff` prints is a real difference: a missing step, a block pasted twice, or a typo.

<details>
<summary>Show the complete file (256 lines)</summary>

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
    """Look up (input, output) prices for a model name, or (None, None) if unknown."""
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
def ask(prompt=None, *, messages=None, system=None, model=HAIKU, max_tokens=512,
        temperature=None, tools=None, tool_choice=None, module="adhoc"):
    """Send one request to Claude, log it, and return the reply."""
    if messages is None:
        messages = [{"role": "user", "content": prompt}]
    args = dict(model=model, max_tokens=max_tokens, messages=messages)
    if system:
        args["system"] = system
    if temperature is not None:
        args["extra_body"] = {"temperature": temperature}   # SDK 1.0+ removed the temperature argument
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

# Each allowed syntax node mapped to the Python function that computes it
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
                   max_tokens=512, max_rounds=10):
    """Run Claude with tools until it answers. Returns (final reply, all replies)."""
    replies = []
    for _ in range(max_rounds):                  # a cap, so a confused model can't loop forever
        resp = ask(messages=history, tools=tools, model=model, system=system,
                   max_tokens=max_tokens, module=module)
        replies.append(resp)
        history.append({"role": "assistant", "content": resp.content})   # keep Claude's turn
        if resp.stop_reason != "tool_use":       # a final answer: we're done
            return resp, replies
        # Claude asked for one or more tools: run each one and collect the results
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
        history.append({"role": "user", "content": results})   # results go back as a user turn
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
    resp = ask(prompt, model=model, max_tokens=512, tools=[tool],
               tool_choice={"type": "tool", "name": tool["name"]}, module=module)
    return tool_input(resp)["label"], resp


# ── Module 3, Step 2 ──
import base64
import io
from pathlib import Path
from PIL import Image


def image_block(path, max_side=1568):
    """Resize an image so its long side is at most max_side px and return a content block."""
    img = Image.open(path)
    img.thumbnail((max_side, max_side))           # keeps aspect ratio; never enlarges
    fmt = "PNG" if img.mode in ("RGBA", "LA", "P") else "JPEG"
    if fmt == "JPEG" and img.mode != "RGB":
        img = img.convert("RGB")
    buf = io.BytesIO()
    img.save(buf, format=fmt)
    return {"type": "image",
            "source": {"type": "base64", "media_type": f"image/{fmt.lower()}",
                       "data": base64.b64encode(buf.getvalue()).decode()}}


# ── Module 3, Step 3 ──
def ask_image(paths, question, *, schema=None, tool_name="record", model=HAIKU,
              max_side=1568, max_tokens=512, module="m3"):
    """Ask about one or more images. With a schema, return structured fields (a dict)."""
    if isinstance(paths, (str, Path)):
        paths = [paths]
    content = [image_block(p, max_side) for p in paths] + [{"type": "text", "text": question}]
    messages = [{"role": "user", "content": content}]
    if schema is None:
        return text_of(ask(messages=messages, model=model, max_tokens=max_tokens, module=module))
    tool = {"name": tool_name, "description": "Record the extracted data.", "input_schema": schema}
    resp = ask(messages=messages, model=model, max_tokens=max_tokens, tools=[tool],
               tool_choice={"type": "tool", "name": tool_name}, module=module)
    return tool_input(resp)


# ── Module 3, Step 5 ──
CHART_SCHEMA = {
    "type": "object",
    "properties": {
        "title": {"type": ["string", "null"]},
        "points": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "series": {"type": "string", "description": "legend name, or 'value' if only one series"},
                    "label": {"type": "string", "description": "x-axis label or category, exactly as printed"},
                    "value": {"type": "number"},
                },
                "required": ["series", "label", "value"],
            },
        },
    },
    "required": ["title", "points"],
}

CHART_PROMPT = ("Extract every data point shown in this chart. Use series names and axis labels "
                "exactly as printed. If a value is not printed, estimate it from the axis.")
```

</details>


## Done when

- [ ] Step 9: error rate measured with one pipeline; worst misses inspected.
- [ ] Step 12: a written decision on unattended vs human-checked image tasks, backed by numbers.

**Next:** [Module 4 — PDFs and documents](../module-04-pdfs-and-documents/README.md)
