# Module 3 — Images (4–5 days)

[← Module 2](../module-02-text-analysis/README.md) · [Syllabus](../README.md) · [Next: Module 4 →](../module-04-pdfs-and-documents/README.md)

**You start with:** Module 2 done — `ask()`, `tool_input()` and the evaluation habit.
**You finish with:** `image_block()` and `ask_image()`; numbers extracted from 20 charts into MongoDB; a measured error rate; and a written decision on which image tasks can run without a human check.

## Key ideas (read once)

- **Images go in as content blocks**, base64-encoded, next to your text question. Supported: JPEG, PNG, GIF, WebP.
- **Bigger images cost more tokens.** Resize before sending; there is no benefit above about 1568 px on the long side.
- **Know what to trust.** Vision is strong at description, layout and printed text; weaker at exact counts, tiny text and precise values read off a chart. You measure this in Step 9.

> Start of session: `cd DataAnalysis_with_LLM && source .venv/bin/activate && docker compose up -d`

---

## Step 1 — Install Pillow and collect 20 charts

```bash
pip install pillow
mkdir -p data/m3/charts
```

Put 20 chart screenshots (bar, line, pie; PNG or JPG) in `data/m3/charts/`. Pick charts whose true numbers you can find, e.g. from reports or dashboards you have the data for.

**Check:** `ls data/m3/charts | wc -l` prints `20`.

## Step 2 — Add `image_block()` to `claude_multimodal.py`

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
from pathlib import Path
from claude_multimodal import image_block
p = sorted(Path('data/m3/charts').iterdir())[0]
b = image_block(p); print(p.name, b['source']['media_type'], len(b['source']['data']), 'chars')"
```

Prints a file name, a media type and a size.

## Step 3 — Add `ask_image()` and describe one chart

Append:

```python
def ask_image(paths, question, *, schema=None, tool_name="record", model=SONNET,
              max_side=1568, max_tokens=2048, module="m3"):
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
from pathlib import Path
from claude_multimodal import ask_image
p = sorted(Path('data/m3/charts').iterdir())[0]
print(ask_image(p, 'Describe this chart: type, title, axes, and the main trend.'))"
```

Prints a sensible description of your first chart.

## Step 4 — See how image size drives cost

Send the same chart at two sizes and compare input tokens. Create `m03_size_cost.py`:

```python
from pathlib import Path
from claude_multimodal import ask, image_block

p = sorted(Path("data/m3/charts").iterdir())[0]
for side in (400, 1568):
    r = ask(messages=[{"role": "user", "content": [image_block(p, side),
            {"type": "text", "text": "What is the chart title?"}]}], max_tokens=50, module="m3")
    print(f"max_side={side}: {r.usage.input_tokens} input tokens -> {r.content[0].text}")
```

```bash
python m03_size_cost.py
```

**Check:** the 1568 px version uses several times more input tokens. Note whether the small version still read the title correctly — small images are cheaper but lose fine text.

## Step 5 — Define the chart schema and extract one chart

Append the schema to `claude_multimodal.py`:

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
from pathlib import Path
from claude_multimodal import ask_image, CHART_SCHEMA, CHART_PROMPT
p = sorted(Path('data/m3/charts').iterdir())[0]
d = ask_image(p, CHART_PROMPT, schema=CHART_SCHEMA, tool_name='record_chart')
print(d['title']); [print(x) for x in d['points'][:5]]"
```

Prints the title and the first data points as dicts.

## Step 6 — Extract all 20 charts into MongoDB

Create `m03_extract.py`:

```python
from pathlib import Path
from claude_multimodal import ask_image, db_rw, CHART_SCHEMA, CHART_PROMPT

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

**Check:** 20 lines of `<file>: N values`, and `python -c "from claude_multimodal import db_ro; print(db_ro.chart_values.count_documents({}))"` prints the total.

## Step 7 — Export 5 charts' values to fill in the truth

Choose 5 charts whose real numbers you know. Export what Claude read so the series and labels match exactly. Create `m03_export_truth.py`:

```python
import sys
import pandas as pd
from claude_multimodal import db_ro

rows = list(db_ro.chart_values.find({"image_file": {"$in": sys.argv[1:]}},
                                    {"_id": 0, "image_file": 1, "series": 1, "label": 1, "value": 1}))
pd.DataFrame(rows).to_csv("data/m3/truth.csv", index=False)
print(len(rows), "rows written to data/m3/truth.csv")
```

```bash
python m03_export_truth.py chart01.png chart02.png chart03.png chart04.png chart05.png
```

Open `data/m3/truth.csv` and **overwrite the `value` column with the true numbers** from your source data. Leave the other columns unchanged.

**Check:** the CSV has rows for your 5 charts and you have corrected every value.

## Step 8 — Load the truth into MongoDB

Create `m03_load_truth.py`:

```python
import pandas as pd
from claude_multimodal import db_rw

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
from claude_multimodal import db_ro

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
from pathlib import Path
from claude_multimodal import ask_image
a, b = sorted(Path('data/m3/charts').iterdir())[:2]
print(ask_image([a, b], 'The first image is chart A, the second chart B. What changed between them?'))"
```

**Check:** the answer refers to both charts by A and B.

## Step 11 — Read text from a screenshot (OCR)

Take a screenshot with printed text (a terminal, an error dialog, a receipt) and save it as `data/m3/screenshot.png`.

```bash
python -c "
from claude_multimodal import ask_image
print(ask_image('data/m3/screenshot.png', 'Transcribe all text in this image exactly, preserving line breaks.'))"
```

**Check:** compare the output to the screenshot line by line; note any misread characters.

## Step 12 — Decide what can run unattended, and commit

Create `notes/m03_decision.md` answering, with your Step 9 numbers: which image tasks you'd run without a human check (e.g. titles, printed text, labeled values) and which need review (e.g. values estimated from bar heights).

```bash
git add claude_multimodal.py m03_*.py notes/m03_decision.md
git commit -m "Module 3: image extraction and error measurement"
git push
```

**Check:** pushed; no images from `data/` in the commit.

## Done when

- [ ] Step 9: error rate measured with one pipeline; worst misses inspected.
- [ ] Step 12: a written decision on unattended vs human-checked image tasks, backed by numbers.

**Next:** [Module 4 — PDFs and documents](../module-04-pdfs-and-documents/README.md)
