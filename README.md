# Multimodal Data Analysis with Claude

An 11-module, self-paced course on analyzing text, images, PDFs, audio, tabular files and logs with Claude, using a local MongoDB as the workbench. About 8 weeks at 1–2 hours a day, plus a capstone.

**Start here → [Module 0 — Setup](module-00-setup/README.md)**

> **Before you start: fork this repo.** Click **Fork** on https://github.com/draju1980/DataAnalysis_with_LLM to make your own copy, then clone your fork (`git clone https://github.com/<your-username>/DataAnalysis_with_LLM.git`). All your course work is committed to your fork. Don't clone the original repo directly: you can't push to it. [Module 0, Step 0](module-00-setup/README.md#step-0--fork-the-course-repo-to-your-github-account) walks through it.

## How to follow this course

- **Fork first, then work in your fork.** Your code and commits live in your own copy of the repo on GitHub.
- **Do the modules in order, and the steps in order.** Every step uses something the step before it created: a file, a function, a collection or a setting. If a step's **Check** fails, fix it before moving on.
- **One shared library.** All reusable functions go into `lib_claude_multimodal.py` in the project root. Each module adds to it, and later modules import what earlier ones built.
- **File naming.** The shared library is the only file with the `lib_` prefix. Every other file you create starts with its module number (`m01_chat.py`, `m02_eval.py`, …), so no two tasks share a file name.
- **Scripts and data.** Each module's runnable scripts (`m01_chat.py`, `m02_eval.py`, …) go in the project root next to `lib_claude_multimodal.py`. Input files go in `data/<module>/`, which git ignores.
- **Course pages vs your work.** The `module-XX-…/README.md` files are the instructions. Your code lives in the project root.

### Start of every session

No module is meant to be done in one sitting. Every module page opens with a **Before you start or resume** section. Run it each time you sit down: it starts your environment, checks that the earlier modules are in place, and prints `done` / `todo` for each step so you can see where you stopped. It also lists which steps are safe to rerun and which are slow or costly to repeat.

The core of it, every time you open a new terminal:

```bash
cd DataAnalysis_with_LLM
source .venv/bin/activate
docker compose up -d
```

To stop for the day, run `docker compose stop` or leave it running; your data stays. Don't use `docker compose down -v` to pause: it deletes the database.

## About the course

You learn to get reliable answers and structured data out of six kinds of input with Claude, and to prove how accurate and how expensive each result is. Every module ends with working code and a pass/fail check.

| | |
| --- | --- |
| Audience | Engineers new to data analysis and AI who are comfortable with the command line, git and Docker |
| Prerequisites | Python basics, git, Docker; a GitHub account (to fork this repo); an Anthropic API account with $10–20 credit |
| Format | Self-paced; each module is a numbered sequence of steps, each with a **Check** |
| Pace | 1–2 hours a day, about 8 weeks for Modules 0–9, then Module 10 and the capstone |
| Environment | Local only: Python, Jupyter, and MongoDB (`mongodb-atlas-local`) in Docker |

Two habits run through every module: **measure accuracy** against answers you know are right, and **track the cost** of every run.

## Learning outcomes

By the end of the course you can:

1. Call the Claude API with tools, streaming and error handling, and track the cost of every call.
2. Turn text, images, PDFs, audio, spreadsheets and logs into structured, validated data with Claude.
3. Build an evaluation set and measure a prompt's accuracy and cost, then compare prompts and models with evidence.
4. Decide which tasks can run unattended and which need a human check, based on measured error rates.
5. Let Claude answer questions by querying data (DuckDB SQL and MongoDB aggregation pipelines) without ever inventing numbers.
6. Combine several formats with vector search and give answers with checkable citations.
7. Guard a Claude pipeline against prompt injection and unsafe queries with least-privilege users and query checks.
8. Package a pipeline with secrets management, logging and evaluation in CI.

## Modules

| # | Module | Duration | Builds on | You build | Done when |
| --- | --- | --- | --- | --- | --- |
| 0 | [Setup](module-00-setup/README.md) | 1 day | — | `.env`, MongoDB users, first notebook | Claude and MongoDB connect; read-only user blocks writes |
| 1 | [Claude API fundamentals](module-01-claude-api-fundamentals/README.md) | 2–3 days | 0 | `lib_claude_multimodal.py`: `ask()`, `log_call()`, tool loop; `m01_chat.py` | CLI chat with running cost and a tool; costs match `llm_calls` |
| 2 | [Text analysis](module-02-text-analysis/README.md) | 1 week | 1 | `classify()`, evaluation scripts, batch run | Prompt versions compared from one query |
| 3 | [Images](module-03-images/README.md) | 4–5 days | 2 | `image_block()`, `ask_image()`, chart extractor | Chart error rate measured |
| 4 | [PDFs and documents](module-04-pdfs-and-documents/README.md) | 4–5 days | 3 | `ask_pdf()`, `extract_pdf_fields()`, folder pipeline | Bad files logged, totals check passing |
| 5 | [Audio and video](module-05-audio-and-video/README.md) | 4–5 days | 4 | `transcribe()`, `label_speakers()`, `analyze_audio()` | One command processes a one-hour recording |
| 6 | [Tabular files](module-06-tabular-files/README.md) | 1 week | 1 | `load_tables()`, `describe_table()`, `ask_data()` | 10 answers verified |
| 7 | [Logs, XML, HTML, YAML](module-07-semi-structured-text/README.md) | 3–4 days | 6 | Log extractor into `log_events` | Count matches grep; 20 records checked |
| 8 | [MongoDB](module-08-mongodb/README.md) | 1 week | 1–7 | `describe_mongo()`, `run_pipeline()`, `ask_mongo()` | 10 `$lookup` answers verified; writes blocked twice |
| 9 | [Combining modalities + vector search](module-09-combining-modalities-vector-search/README.md) | 1 week | 4, 5, 7, 8 | Chunks, vector index, `search_chunks()`, assistant | 10 answers with checkable sources |
| 10 | [Production and security](module-10-production-and-security/README.md) | Ongoing | 0–9 | Container, CI eval gate, backups, threat model | Worse prompt fails CI; injections have no effect |
| — | [Capstone](capstone/README.md) | 2 weeks | All | One real multi-format project | Accuracy, cost and threat model written up |

## How the local database helps you learn faster

Every module saves what Claude produces into MongoDB, so checking accuracy, comparing prompts or adding up cost takes one query. `docker compose down -v` resets it in seconds (then repeat Module 0 Steps 9–12).

| Module | Collection | What one query tells you |
| --- | --- | --- |
| 1 | `llm_calls` | Cost and latency by model and module |
| 2 | `eval_items`, `eval_results`, `predictions` | Accuracy and cost per prompt version and model |
| 3 | `chart_values`, `chart_truth` | Error rate and the worst misses |
| 4 | `invoices`, `file_errors` | Invoices whose lines don't add up; files that failed |
| 5 | `transcript_segments`, `action_items` | Who committed to what, and when |
| 7 | `log_events` | Record count vs grep; errors by service and hour |
| 9 | `chunks` | The sources behind each answer |

## Weekly schedule

| Week | Modules |
| --- | --- |
| 1 | 0 Setup · 1 Claude API fundamentals · start 2 |
| 2 | 2 Text analysis · start 3 |
| 3 | 3 Images · 4 PDFs and documents |
| 4 | 5 Audio and video · start 6 |
| 5 | 6 Tabular files · start 7 |
| 6 | 7 Logs, XML, HTML, YAML · 8 MongoDB |
| 7 | 8 MongoDB · 9 Combining modalities |
| 8 | 9 Combining modalities · start 10 and the capstone |
| 9–10 | 10 Production and security · Capstone |

## Assessment and completion

There are no quizzes or grades. A module is complete when its **Done when** checklist passes. The course is complete when all eleven modules pass and the [capstone](capstone/README.md) is written up with accuracy, cost per run and a threat model.

## Tools

| Tool | Used for | First needed |
| --- | --- | --- |
| Anthropic API, model `claude-haiku-4-5-20251001`, `max_tokens=512` | All analysis | Module 0 |
| Python, Jupyter, python-dotenv | Code, notebooks, secrets from `.env` | Module 0 |
| Docker, MongoDB `mongodb-atlas-local`, mongosh, pymongo | Local database with Vector Search | Module 0 |
| Pillow | Resizing images | Module 3 |
| pypdf | Counting, splitting and reading PDF pages | Module 4 |
| faster-whisper, ffmpeg | Transcription and video frames (the one non-Claude model) | Module 5 |
| pandas, DuckDB, openpyxl, pyarrow | Tabular files and SQL on files | Module 6 |
| grep, jq | Pre-filtering logs | Module 7 |
| Voyage AI (optional) | Embeddings; skip it to stay Claude-only with `$search` | Module 9 |
| MongoDB Database Tools | Backups (`mongodump`, `mongorestore`) | Module 10 |

## Progress checklist

- [ ] Module 0 — Claude and MongoDB connect; read-only user blocks writes; no secrets in git
- [ ] Module 1 — CLI chat with running cost and a tool; totals match `llm_calls`
- [ ] Module 2 — Evaluation runs in MongoDB; prompt versions compared
- [ ] Module 3 — Chart values extracted; error rate measured
- [ ] Module 4 — PDF folder in `invoices`; bad files logged; totals check passing
- [ ] Module 5 — One-hour recording processed in one command
- [ ] Module 6 — Three-format analysis; 10 answers verified
- [ ] Module 7 — Logs in `log_events`; count and spot checks pass
- [ ] Module 8 — 10 `$lookup` answers verified; writes blocked at checker and user
- [ ] Module 9 — Assistant answers with checkable citations
- [ ] Module 10 — Containerized job, CI eval gate, backups, threat model
- [ ] Capstone — built and written up

## Repository layout

```
README.md                         ← this syllabus
module-00-setup/README.md         ← instructions, one folder per module
module-01-claude-api-fundamentals/README.md
…
module-10-production-and-security/README.md
capstone/README.md

lib_claude_multimodal.py          ← your shared library (built from Module 1)
m01_chat.py, m02_eval.py, …       ← your module scripts
data/                             ← your input files (git-ignored)
```
