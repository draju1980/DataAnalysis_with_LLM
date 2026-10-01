# Module 6 — Tabular files: CSV, TSV, Excel, JSON, Parquet (1 week)

[← Module 5](../module-05-audio-and-video/README.md) · [Syllabus](../README.md) · [Next: Module 7 →](../module-07-semi-structured-text/README.md)

**You start with:** Module 1 done — `ask()`, `run_with_tools()`, `text_of()` (Modules 2–5 aren't required for this one).
**You finish with:** `load_tables()`, `describe_table()`, `run_sql()` and `ask_data()`; three files in different formats queryable together with SQL; and 10 questions answered by Claude, every number checked against your own result.

## Key ideas (read once)

- **Profile, don't paste.** Claude never sees the whole table. It sees column names, types, missing-value counts and a few sample rows, then writes SQL to compute answers. Numbers are calculated, not guessed.
- **DuckDB** runs SQL directly on CSV, Excel-via-pandas, JSON and Parquet files — no database server, no loading step.
- **The rule for Claude:** every number in an answer must come from a query result in that conversation.

> Start of session: `cd DataAnalysis_with_LLM && source .venv/bin/activate && docker compose up -d`

---

## Step 1 — Install the tabular packages and add three files

```bash
pip install duckdb openpyxl pyarrow
mkdir -p data/m6
```

Put three related files in **different formats** in `data/m6/`, for example:

| File | Example content |
| --- | --- |
| `trades.csv` | one row per trade: date, symbol, side, quantity, price |
| `targets.xlsx` | monthly targets per symbol or desk |
| `export.json` | an API export, e.g. instrument details |

They should share at least one column you can join on (like `symbol` or month).

**Check:** `ls data/m6` shows your three files.

## Step 2 — Look at the raw files and note the quirks

Before loading anything, look at what you have:

```bash
head -5 data/m6/trades.csv
python -c "
import pandas as pd
x = pd.ExcelFile('data/m6/targets.xlsx'); print('sheets:', x.sheet_names)
print(pd.read_excel(x, sheet_name=x.sheet_names[0], header=None).head(8))"
python -c "import json; d=json.load(open('data/m6/export.json')); print(type(d).__name__, list(d)[:5] if isinstance(d, dict) else d[:1])"
```

Write down:

- **CSV:** the date format (e.g. `30/09/2026` vs `2026-09-30`), and any numbers with commas or currency signs.
- **Excel:** which sheet holds the data and which row the headers are on (0-based; with a title row above them it's often `1` or `2`).
- **JSON:** whether it's a list of records, or a dict with the records under a key such as `"data"`.

**Check:** you've noted the sheet name, header row and JSON shape.

## Step 3 — Add `load_tables()` and load all three files

Append to `claude_multimodal.py`:

```python
import json


def load_tables(spec):
    """Load files into an in-memory DuckDB. spec = {table_name: path or (path, options)}."""
    import duckdb
    import pandas as pd
    con = duckdb.connect()
    for name, src in spec.items():
        path, opts = (src, {}) if isinstance(src, str) else src
        ext = Path(path).suffix.lower()
        if ext in (".csv", ".tsv"):
            extra = "".join(f", {k}={v!r}" for k, v in opts.items())   # e.g. types={...}
            con.execute(f"CREATE VIEW {name} AS SELECT * FROM read_csv_auto('{path}'{extra})")
        elif ext == ".parquet":
            con.execute(f"CREATE VIEW {name} AS SELECT * FROM read_parquet('{path}')")
        elif ext in (".xlsx", ".xls"):
            con.register(name, pd.read_excel(path, **opts))
        elif ext == ".json":
            data = json.loads(Path(path).read_text())
            records = data[opts["records_key"]] if "records_key" in opts else data
            con.register(name, pd.json_normalize(records))     # flattens nested JSON
        else:
            raise ValueError(f"unsupported file type: {path}")
    return con
```

Create `m06_tables.py` with your file list. Use the sheet, header row and JSON key you noted in Step 2:

```python
from claude_multimodal import load_tables

SPEC = {
    "trades": "data/m6/trades.csv",
    "targets": ("data/m6/targets.xlsx", {"sheet_name": "2026", "header": 2}),
    "instruments": ("data/m6/export.json", {"records_key": "data"}),
}


def open_tables():
    return load_tables(SPEC)


if __name__ == "__main__":
    con = open_tables()
    for name in SPEC:
        print(name, con.execute(f"SELECT count(*) FROM {name}").fetchone()[0], "rows")
```

```bash
python m06_tables.py
```

**Check:** prints a row count for each of the three tables.

## Step 4 — Add `describe_table()` and profile each table

Append to `claude_multimodal.py`:

```python
def describe_table(con, name, n=5):
    """Columns with types and missing counts, row count, and sample rows."""
    cols = con.execute(f"DESCRIBE {name}").fetchall()             # (name, type, ...)
    rows = con.execute(f"SELECT count(*) FROM {name}").fetchone()[0]
    missing = con.execute("SELECT " + ", ".join(f'count(*) - count("{c[0]}")' for c in cols)
                          + f" FROM {name}").fetchone()
    sample = con.execute(f"SELECT * FROM {name} LIMIT {n}").df().to_string(index=False)
    lines = [f"Table {name}: {rows} rows"]
    lines += [f"- {c[0]} ({c[1]}), {m} missing" for c, m in zip(cols, missing)]
    return "\n".join(lines) + f"\nSample rows:\n{sample}"
```

```bash
python -c "
from m06_tables import open_tables, SPEC
from claude_multimodal import describe_table
con = open_tables()
for t in SPEC: print(describe_table(con, t), '\n')"
```

**Check:** every column has the type you expect. If a date column shows `VARCHAR`, fix it in the next step.

## Step 5 — Fix type quirks with a clean view (only if Steps 3–4 showed problems)

DuckDB guesses each column's type from a sample of rows. With **mixed date formats** (`2026-09-03` in one row, `04/09/2026` in another) it either keeps the column as text (`VARCHAR`) or guesses `DATE` and then fails at query time with *Could not convert string … to 'DATE'*.

Fix it in two moves in `m06_tables.py`:

1. Load the column as text, so nothing fails while loading. Change the trades entry in `SPEC`:

   ```python
   "trades": ("data/m6/trades.csv", {"types": {"trade_date": "VARCHAR"}}),
   ```

2. Parse it yourself in a clean view. Replace `open_tables()` with:

   ```python
   def open_tables():
       con = load_tables(SPEC)
       con.execute("""CREATE OR REPLACE VIEW trades_clean AS
                      SELECT * REPLACE (coalesce(try_strptime(trade_date, '%d/%m/%Y'),
                                                 try_strptime(trade_date, '%Y-%m-%d'))::DATE AS trade_date)
                      FROM trades""")
       return con
   ```

   `try_strptime` returns `NULL` instead of failing, and `coalesce` takes the first format that works. List every format you saw in Step 2.

Use `trades_clean` instead of `trades` from now on.

**Check:** rerun the Step 4 command with `trades_clean` added; `trade_date` shows type `DATE` and `0 missing` (a non-zero missing count means a format you haven't listed).

## Step 6 — Add `run_sql()` with a read-only check

Claude's SQL must only read. Append:

```python
def run_sql(con, sql, max_rows=200):
    """Run one SELECT (or WITH … SELECT) and return the result as text."""
    s = sql.strip().rstrip(";")
    if ";" in s or not s.lower().startswith(("select", "with")):
        raise ValueError("only one SELECT (or WITH … SELECT) statement is allowed")
    df = con.execute(s).df()
    note = f"\n({len(df)} rows; showing the first {max_rows})" if len(df) > max_rows else ""
    return df.head(max_rows).to_string(index=False) + note
```

**Check:**

```bash
python -c "
from m06_tables import open_tables
from claude_multimodal import run_sql
con = open_tables()
print(run_sql(con, 'SELECT count(*) AS n FROM trades'))
try: run_sql(con, 'DROP VIEW trades')
except ValueError as e: print('blocked:', e)"
```

Prints the count, then `blocked: only one SELECT …`.

## Step 7 — Add `ask_data()` and ask one question

Append:

```python
SQL_TOOL = {
    "name": "run_sql",
    "description": "Run one read-only DuckDB SQL query and get the result table.",
    "input_schema": {"type": "object", "properties": {"sql": {"type": "string"}}, "required": ["sql"]},
}


def ask_data(con, question, tables, *, model=SONNET):
    """Answer a question by letting Claude query the tables. Returns (answer, sql_list)."""
    schema = "\n\n".join(describe_table(con, t) for t in tables)
    system = ("You answer questions about data by querying it with the run_sql tool (DuckDB SQL). "
              "Every number in your answer must come from a query result in this conversation; "
              "never estimate or recall numbers. If the data can't answer the question, say so. "
              'Quote column names that contain dots, e.g. "meta.cls".\n\n'
              "Tables:\n" + schema)
    history = [{"role": "user", "content": question}]
    resp, replies = run_with_tools(history, [SQL_TOOL], {"run_sql": lambda sql: run_sql(con, sql)},
                                   system=system, model=model, module="m6")
    sqls = [b.input["sql"] for r in replies for b in r.content if b.type == "tool_use"]
    return text_of(resp), sqls
```

**Check:**

```bash
python -c "
from m06_tables import open_tables
from claude_multimodal import ask_data
answer, sqls = ask_data(open_tables(), 'How many trades are there in total?', ['trades_clean', 'targets', 'instruments'])
print(answer); print(sqls)"
```

Prints `[tool] run_sql …`, an answer, and the SQL used. The number matches Step 3's row count. (If you skipped Step 5, use `'trades'` instead of `'trades_clean'` here and in Step 9.)

## Step 8 — Write 10 questions and your own answers

Create `data/m6/questions.txt` with 10 questions, one per line. Include at least three that need **two or more files**, e.g.:

```
What was the total traded value per symbol in September?
Which symbols beat their monthly target in August, and by how much?
What is the average trade size for instruments in the "equity" asset class?
```

Now compute each answer **yourself**, with your own SQL in DuckDB or pandas, and write them in `notes/m06_answers.md`. Example for one question:

```bash
python -c "
from m06_tables import open_tables
con = open_tables()
print(con.execute('''SELECT symbol, sum(quantity*price) AS value FROM trades_clean
                     WHERE month(trade_date)=9 GROUP BY symbol ORDER BY value DESC''').df())"
```

**Check:** `notes/m06_answers.md` has 10 answers you computed yourself.

## Step 9 — Let Claude answer all 10

Create `m06_ask.py`:

```python
from pathlib import Path
from m06_tables import open_tables
from claude_multimodal import ask_data

FENCE = "`" * 3                       # a Markdown code fence
TABLES = ["trades_clean", "targets", "instruments"]
con = open_tables()
out = ["# Module 6 — Claude's answers", ""]
for n, q in enumerate(Path("data/m6/questions.txt").read_text().splitlines(), 1):
    if not q.strip():
        continue
    answer, sqls = ask_data(con, q, TABLES)
    out += [f"## {n}. {q}", "", answer, "", "SQL used:", ""] + [f"{FENCE}sql\n{s}\n{FENCE}" for s in sqls] + [""]
    print(f"{n}. done")
Path("notes/m06_results.md").write_text("\n".join(out))
print("wrote notes/m06_results.md")
```

```bash
python m06_ask.py
```

**Check:** `notes/m06_results.md` has 10 answers, each with the SQL that produced it.

## Step 10 — Compare and fix

Put `notes/m06_answers.md` and `notes/m06_results.md` side by side. For each mismatch, read Claude's SQL to find the cause (wrong join, wrong date filter, misread column), then fix the **input**, not the answer: rename a confusing column in a clean view, or add a sentence to the `system` text in `ask_data()` explaining it. Rerun Step 9.

**Check:** all 10 answers match yours, and every number in `m06_results.md` appears in a query result.

## Step 11 — Commit

```bash
git add claude_multimodal.py m06_*.py notes/m06_answers.md notes/m06_results.md
git commit -m "Module 6: tabular files, DuckDB, ask_data"
git push
```

**Check:** pushed; your data files are not in the commit.

## Done when

- [ ] Step 10: all 10 answers match the results you calculated yourself.
- [ ] Step 10: Claude never states a number it didn't query.

**Next:** [Module 7 — Logs, XML, HTML, YAML](../module-07-semi-structured-text/README.md)
