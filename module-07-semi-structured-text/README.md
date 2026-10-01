# Module 7 — Semi-structured text: logs, XML, HTML, YAML (3–4 days)

[← Module 6](../module-06-tabular-files/README.md) · [Syllabus](../README.md) · [Next: Module 8 →](../module-08-mongodb/README.md)

**You start with:** Module 6 done — `ask()`, `tool_input()`, the forced-tool pattern and the habit of checking counts.
**You finish with:** `extract_log_records()` and `ask_text_file()`; a log file turned into the `log_events` collection; a record count that matches `grep`; 20 records spot-checked; and questions answered directly from small YAML/XML/HTML files.

## Key ideas (read once)

- **Some structure, no fixed columns.** Logs, XML, HTML and YAML need to become clean records before you can query them.
- **Use ordinary tools first.** `grep`, `jq`, `head` and parsers cut the input down so Claude sees only what matters — cheaper and more accurate.
- **One record per event.** A stack trace spread over 20 lines is one event, which is exactly what simple line-based parsing gets wrong and Claude gets right.
- **Verify by counting.** The number of records must match an independent count from `grep`.

> Start of session: `cd DataAnalysis_with_LLM && source .venv/bin/activate && docker compose up -d`

---

## Step 1 — Get a log file

```bash
mkdir -p data/m7
```

Use one of:

- An application or CI log: copy it to `data/m7/app.log`.
- Kubernetes events — convert the JSON to one line per event with `jq`:

  ```bash
  kubectl get events -A -o json | jq -r '.items[] |
    "\(.lastTimestamp) \(.involvedObject.namespace)/\(.involvedObject.name) \(.type) \(.reason): \(.message)"' \
    > data/m7/app.log
  ```

**Check:** `wc -l data/m7/app.log` shows the line count, and `head data/m7/app.log` looks like log lines.

## Step 2 — Cut it down to a sample

Work on a 1,000-line sample while you build; run bigger files only after it works.

```bash
head -n 1000 data/m7/app.log > data/m7/sample.log
```

**Check:** `wc -l data/m7/sample.log` prints `1000`.

## Step 3 — Count the events with grep

Find the pattern that starts every event — usually a timestamp at the beginning of the line. For ISO timestamps like `2026-09-30T14:03:22Z`:

```bash
grep -cE '^[0-9]{4}-[0-9]{2}-[0-9]{2}' data/m7/sample.log
```

Adjust the pattern to your log's format. Lines that don't match (stack trace lines, wrapped messages) belong to the event above them.

Write the number down — it's the target count for Step 6.

**Check:** you have one number, and `grep -vcE '<your pattern>' data/m7/sample.log` shows how many continuation lines there are.

## Step 4 — Add `extract_log_records()` and test it on 50 lines

Append to `claude_multimodal.py`:

```python
LOG_TOOL = {
    "name": "record_events",
    "description": "Record one record per log event.",
    "input_schema": {"type": "object", "properties": {"records": {"type": "array", "items": {
        "type": "object",
        "properties": {
            "line": {"type": "integer", "description": "line number where the event starts"},
            "ts": {"type": "string", "description": "ISO 8601 with timezone, e.g. 2026-09-30T14:03:22Z"},
            "service": {"type": "string"},
            "severity": {"type": "string", "enum": ["DEBUG", "INFO", "WARN", "ERROR", "FATAL"]},
            "message": {"type": "string"},
            "attrs": {"type": "object", "description": "any other fields, e.g. namespace, pod, request_id"},
        },
        "required": ["line", "ts", "service", "severity", "message"]}}},
        "required": ["records"]},
}


def extract_log_records(lines, first_line_no=1, model=HAIKU):
    """Turn raw log lines into records (one per event, multi-line events merged)."""
    numbered = "\n".join(f"{first_line_no + i}: {line.rstrip()}" for i, line in enumerate(lines))
    prompt = ("Turn these numbered log lines into records, one per log event. A stack trace or "
              "message continuing over several lines is ONE event. Use 'unknown' for a missing "
              "service. Put any other fields in attrs.\n"
              f"<log>\n{numbered}\n</log>")
    resp = ask(prompt, system=DOC_RULE, model=model, max_tokens=12000, tools=[LOG_TOOL],
               tool_choice={"type": "tool", "name": "record_events"}, module="m7")
    return tool_input(resp)["records"]
```

`DOC_RULE` (from Module 4) tells Claude the log is data, not instructions — log lines can contain attacker-controlled text.

**Check:**

```bash
python -c "
from claude_multimodal import extract_log_records
lines = open('data/m7/sample.log').readlines()[:50]
recs = extract_log_records(lines)
print(len(recs), 'records'); [print(r) for r in recs[:3]]"
```

Prints a record count (compare with `head -50 data/m7/sample.log | grep -cE '<your pattern>'`) and well-formed records.

## Step 5 — Extract the whole sample into `log_events`

Create `m07_extract.py`. It sends 100 lines at a time and converts `ts` to a real date.

```python
import sys
from datetime import datetime
from pathlib import Path
from claude_multimodal import extract_log_records, db_rw

path = Path(sys.argv[1])
lines = path.read_text(errors="replace").splitlines()
CHUNK = 100

db_rw.log_events.delete_many({"source_file": path.name})        # rerun-safe
total = 0
for start in range(0, len(lines), CHUNK):
    records = extract_log_records(lines[start:start + CHUNK], first_line_no=start + 1)
    for r in records:
        try:
            r["ts"] = datetime.fromisoformat(r["ts"].replace("Z", "+00:00"))
        except ValueError:
            r["ts"] = None                                      # keep the record, flag the date
        r["source_file"] = path.name
    if records:
        db_rw.log_events.insert_many(records)
    total += len(records)
    print(f"lines {start + 1}-{start + len(lines[start:start + CHUNK])}: {len(records)} records")
print("total records:", total)
```

```bash
python m07_extract.py data/m7/sample.log
```

A limitation to know: an event that straddles a 100-line boundary can be split in two. If your log has long stack traces, the count in Step 6 may be off by a few — look at the records near line 100, 200, …

**Check:** prints records per chunk and a total.

## Step 6 — Check the count against grep

```bash
python -c "
from claude_multimodal import db_ro
print('records:', db_ro.log_events.count_documents({'source_file': 'sample.log'}))
print('missing ts:', db_ro.log_events.count_documents({'source_file': 'sample.log', 'ts': None}))"
```

**Check:** `records` equals your Step 3 number (or differs only by events split at chunk boundaries), and `missing ts` is 0 or explained.

## Step 7 — Spot-check 20 random records

Create `m07_spotcheck.py`. It shows each record next to the original line it came from.

```python
from claude_multimodal import db_ro

lines = open("data/m7/sample.log", errors="replace").read().splitlines()
for r in db_ro.log_events.aggregate([{"$match": {"source_file": "sample.log"}},
                                      {"$sample": {"size": 20}}]):
    print(f"line {r['line']}: {lines[r['line'] - 1][:150]}")
    print(f"  -> {r['ts']} | {r['service']} | {r['severity']} | {r['message'][:100]} | {r.get('attrs')}\n")
```

```bash
python m07_spotcheck.py
```

**Check:** in all 20, timestamp, service, severity and message match the original line.

## Step 8 — Query the events

```bash
python -c "
from claude_multimodal import db_ro
for r in db_ro.log_events.aggregate([
    {'\$match': {'source_file': 'sample.log', 'ts': {'\$ne': None}}},
    {'\$group': {'_id': {'hour': {'\$dateTrunc': {'date': '\$ts', 'unit': 'hour'}},
                         'service': '\$service', 'severity': '\$severity'},
                 'count': {'\$sum': 1}}},
    {'\$sort': {'_id.hour': 1, 'count': -1}}]):
    print(r['_id']['hour'], r['_id']['service'], r['_id']['severity'], r['count'])"
```

Fields inside `attrs` are queryable too, e.g. `{'attrs.namespace': 'prod'}`.

**Check:** one line per hour, service and severity, with counts that add up to Step 6's total.

## Step 9 — Ask questions about small YAML, XML or HTML files directly

Small files don't need extraction; send them whole. Append to `claude_multimodal.py`:

```python
def ask_text_file(path, question, *, model=SONNET, max_chars=200_000):
    """Answer a question about a small text file (YAML, XML, HTML, log…)."""
    text = Path(path).read_text(errors="replace")
    if len(text) > max_chars:
        raise ValueError(f"{path} is {len(text)} characters; pre-filter it with grep/jq/yq first")
    prompt = f'<file name="{Path(path).name}">\n{text}\n</file>\n\n{question}'
    return text_of(ask(prompt, system=DOC_RULE, model=model, max_tokens=1500, module="m7"))
```

Try it on the Compose file you wrote in Module 0:

```bash
python -c "
from claude_multimodal import ask_text_file
print(ask_text_file('docker-compose.yml', 'Which ports are published, on which host interface, and which data is persisted?'))"
```

**Check:** the answer says port 27017 is published on 127.0.0.1 only and names the three volumes.

## Step 10 — Run the full log (optional) and commit

If the sample worked, run the full file: `python m07_extract.py data/m7/app.log` and repeat Steps 6–7 for `app.log`.

```bash
git add claude_multimodal.py m07_*.py
git commit -m "Module 7: log extraction, count and spot checks, small-file questions"
git push
```

**Check:** pushed; no log files from `data/` in the commit.

## Done when

- [ ] Step 6: the record count matches the `grep` count.
- [ ] Step 7: 20 spot-checked records are correct.
- [ ] Step 9: a question about a YAML/XML/HTML file answered correctly.

**Next:** [Module 8 — MongoDB](../module-08-mongodb/README.md)
