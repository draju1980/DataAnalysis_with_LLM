# Module 9 — Combining modalities and vector search (1 week)

[← Module 8](../module-08-mongodb/README.md) · [Syllabus](../README.md) · [Next: Module 10 →](../module-10-production-and-security/README.md)

**You start with:** Modules 4, 5, 7 and 8 done — PDFs in `data/m4/pdfs`, `transcript_segments`, `log_events`, and `run_pipeline()`.
**You finish with:** a `chunks` collection built from PDFs, transcripts and logs; a search index; `search_chunks()` and `answer_with_sources()`; and an assistant that picks between searching documents and querying data, citing a source you can open for every answer.

## Key ideas (read once)

- **Retrieval (RAG).** Split content into chunks, find the chunks most relevant to a question, and put only those in the prompt.
- **Two ways to find relevant chunks:**
  - **Route A — vector search (by meaning).** Each chunk becomes an *embedding* (a list of numbers representing its meaning). Anthropic doesn't offer an embedding model and recommends Voyage AI — a second, non-Claude model.
  - **Route B — full-text search (by words), Claude-only.** MongoDB's `$search` matches words; Claude first rewrites the question into search terms. It misses paraphrases but needs no extra model.
- **Citations.** Every chunk keeps its source file plus page, timestamp or line, so every answer points back to something you can open.
- **Retrieval isn't always better.** A small document that fits in the prompt often answers better sent whole (Step 11).

> Start of session: `cd DataAnalysis_with_LLM && source .venv/bin/activate && docker compose up -d`

---

## Step 1 — Choose your route

Decide now; Steps 2, 4, 5 and 6 depend on it.

| | Route A — vector | Route B — text |
| --- | --- | --- |
| Finds | Similar meaning ("revenue fell" ≈ "sales dropped") | Matching words |
| Needs | A Voyage AI API key (non-Claude model) | Nothing extra |

Append to `claude_multimodal.py`, with your choice:

```python
SEARCH_ROUTE = "vector"        # "vector" (Route A) or "text" (Route B)
```

**Check:** `python -c "from claude_multimodal import SEARCH_ROUTE; print(SEARCH_ROUTE)"` prints your choice.

## Step 2 — Route A only: set up Voyage AI

Skip this step on Route B.

1. Create an API key at voyageai.com.
2. Install the client: `pip install voyageai`
3. Add a line to `.env`: `VOYAGE_API_KEY=pa-...`, and the empty name to `.env.example`: `echo 'VOYAGE_API_KEY=' >> .env.example`

**Check:**

```bash
python -c "
import claude_multimodal, voyageai          # importing claude_multimodal loads .env
v = voyageai.Client().embed(['hello'], model='voyage-3.5', input_type='document').embeddings[0]
print(len(v), 'dimensions')"
```

Prints `1024 dimensions`.

## Step 3 — Build the `chunks` collection

Append to `claude_multimodal.py`:

```python
def build_chunks():
    """Rebuild chunks from PDFs (per page), transcripts (8 segments) and WARN+ log events."""
    from pypdf import PdfReader
    docs = []
    for rec in db_ro.transcript_segments.distinct("recording"):
        segs = list(db_ro.transcript_segments.find({"recording": rec}).sort("i"))
        for k in range(0, len(segs), 8):
            window = segs[k:k + 8]
            docs.append({"modality": "transcript", "source_file": rec, "start_s": window[0]["start_s"],
                         "content": " ".join(f"{s.get('speaker') or '?'}: {s['text']}" for s in window)})
    for path in sorted(Path("data/m4/pdfs").glob("*.pdf")):
        try:
            reader = PdfReader(path)
        except Exception:
            continue                                    # corrupt files were logged in Module 4
        for n, page in enumerate(reader.pages, 1):
            text = (page.extract_text() or "").strip()
            if len(text) > 50:                          # scanned pages have no text layer
                docs.append({"modality": "pdf", "source_file": path.name, "page": n, "content": text[:4000]})
    for e in db_ro.log_events.find({"severity": {"$in": ["WARN", "ERROR", "FATAL"]}}):
        docs.append({"modality": "log", "source_file": e["source_file"], "line": e.get("line"),
                     "content": f"{e['ts']} {e['service']} {e['severity']}: {e['message']}"})
    db_rw.chunks.delete_many({})
    if docs:
        db_rw.chunks.insert_many(docs)
    return len(docs)


def cite(chunk):
    """Human-readable source of a chunk: file plus page, time or line."""
    if chunk.get("page"):
        return f"{chunk['source_file']} p.{chunk['page']}"
    if chunk.get("start_s") is not None:
        return f"{chunk['source_file']} at {fmt_ts(chunk['start_s'])}"
    if chunk.get("line"):
        return f"{chunk['source_file']} line {chunk['line']}"
    return chunk["source_file"]
```

```bash
python -c "
from claude_multimodal import build_chunks, db_ro
print(build_chunks(), 'chunks')
for r in db_ro.chunks.aggregate([{'\$group': {'_id': '\$modality', 'n': {'\$sum': 1}}}]): print(r)"
```

**Check:** prints a total and a count for `pdf`, `transcript` and `log`.

## Step 4 — Route A only: embed the chunks

Skip this step on Route B. Append:

```python
def embed(texts, input_type):
    """Voyage embeddings in batches of 128. input_type is 'document' or 'query'."""
    import voyageai
    vo = voyageai.Client()
    vectors = []
    for k in range(0, len(texts), 128):
        vectors += vo.embed(texts[k:k + 128], model="voyage-3.5", input_type=input_type).embeddings
    return vectors


def embed_chunks():
    """Add an embedding to every chunk that doesn't have one yet."""
    todo = list(db_rw.chunks.find({"embedding": {"$exists": False}}, {"content": 1}))
    for doc, vec in zip(todo, embed([d["content"] for d in todo], "document")):
        db_rw.chunks.update_one({"_id": doc["_id"]}, {"$set": {"embedding": vec}})
    return len(todo)
```

```bash
python -c "from claude_multimodal import embed_chunks; print(embed_chunks(), 'chunks embedded')"
```

**Check:** the number equals Step 3's total, and `python -c "from claude_multimodal import db_ro; print(db_ro.chunks.count_documents({'embedding': {'\$exists': False}}))"` prints `0`.

## Step 5 — Create the search index and wait until it's ready

Append:

```python
def create_chunk_index():
    """Create the index for SEARCH_ROUTE and wait until it can be queried."""
    from pymongo.operations import SearchIndexModel
    if SEARCH_ROUTE == "vector":
        name, model = "chunks_vec", SearchIndexModel(name="chunks_vec", type="vectorSearch", definition={
            "fields": [{"type": "vector", "path": "embedding", "numDimensions": 1024, "similarity": "cosine"},
                       {"type": "filter", "path": "modality"}]})
    else:
        name, model = "chunks_text", SearchIndexModel(name="chunks_text", type="search", definition={
            "mappings": {"dynamic": False, "fields": {
                "content": {"type": "string"}, "modality": {"type": "token"}}}})
    if not any(i["name"] == name for i in db_rw.chunks.list_search_indexes()):
        db_rw.chunks.create_search_index(model)
    while not next(i for i in db_rw.chunks.list_search_indexes() if i["name"] == name).get("queryable"):
        print("index building…")
        time.sleep(5)
    return name
```

```bash
python -c "from claude_multimodal import create_chunk_index; print(create_chunk_index(), 'is ready')"
```

If you get a "not authorized" error, create the index once as admin with mongosh (paste password 1), then run the command above again — it finds the index and waits for it:

```bash
# Route A
mongosh "mongodb://admin@127.0.0.1:27017/admin?directConnection=true" --quiet --eval '
  db.getSiblingDB("course").chunks.createSearchIndex("chunks_vec", "vectorSearch",
    {fields: [{type: "vector", path: "embedding", numDimensions: 1024, similarity: "cosine"},
              {type: "filter", path: "modality"}]})'
# Route B
mongosh "mongodb://admin@127.0.0.1:27017/admin?directConnection=true" --quiet --eval '
  db.getSiblingDB("course").chunks.createSearchIndex("chunks_text", "search",
    {mappings: {dynamic: false, fields: {content: {type: "string"}, modality: {type: "token"}}}})'
```

**Check:** prints `chunks_vec is ready` (Route A) or `chunks_text is ready` (Route B).

## Step 6 — Add `search_chunks()` and test it

Append:

```python
def search_chunks(question, modality=None, k=5):
    """Find the k most relevant chunks (optionally only pdf / transcript / log)."""
    project = {"$project": {"_id": 0, "modality": 1, "source_file": 1, "page": 1, "start_s": 1,
                            "line": 1, "content": 1}}
    if SEARCH_ROUTE == "vector":
        stage = {"index": "chunks_vec", "path": "embedding", "queryVector": embed([question], "query")[0],
                 "numCandidates": 100, "limit": k}
        if modality:
            stage["filter"] = {"modality": modality}
        pipeline = [{"$vectorSearch": stage}, project]
    else:
        terms = text_of(ask("Rewrite this question as 3-8 search keywords, space-separated, "
                            f"nothing else:\n{question}", max_tokens=1024, module="m9"))
        search = {"index": "chunks_text", "compound": {"must": [{"text": {"query": terms, "path": "content"}}]}}
        if modality:
            search["compound"]["filter"] = [{"equals": {"path": "modality", "value": modality}}]
        pipeline = [{"$search": search}, {"$limit": k}, project]
    return list(db_ro.chunks.aggregate(pipeline))
```

**Check:** ask about something you know is in one of your sources:

```bash
python -c "
from claude_multimodal import search_chunks, cite
for h in search_chunks('What did the CFO say about pricing?'): print(cite(h), '|', h['content'][:100])"
```

Prints 5 sources; the right one is among them.

## Step 7 — Add `answer_with_sources()`

Append:

```python
def answer_with_sources(question, modality=None, k=8, model=HAIKU):
    """Answer from retrieved chunks only, citing them as [1], [2]…"""
    hits = search_chunks(question, modality, k)
    sources = "\n\n".join(f"[{n}] ({h['modality']}, {cite(h)})\n{h['content']}" for n, h in enumerate(hits, 1))
    prompt = ("Answer the question using only the numbered sources. Put the source number in "
              "brackets after each claim, like [2]. If the sources don't answer it, say so.\n"
              f"<sources>\n{sources}\n</sources>\nQuestion: {question}")
    answer = text_of(ask(prompt, system=DOC_RULE, model=model, max_tokens=1024, module="m9"))
    return answer + "\n\nSources:\n" + "\n".join(f"[{n}] {cite(h)}" for n, h in enumerate(hits, 1))
```

**Check:**

```bash
python -c "
from claude_multimodal import answer_with_sources
print(answer_with_sources('What did the CFO say about pricing?'))"
```

Prints an answer with `[n]` markers and a source list with files, pages, times or lines.

## Step 8 — Build the assistant that picks the right tool

Some questions need documents (Step 7), others need numbers from collections (Module 8). Create `m09_assistant.py`:

```python
from claude_multimodal import (run_with_tools, run_pipeline, search_chunks, cite, describe_mongo,
                               text_of, PIPELINE_TOOL, DOC_RULE, HAIKU)

SEARCH_TOOL = {
    "name": "search_documents",
    "description": "Search PDF pages, call transcripts and log events. Returns passages with their sources.",
    "input_schema": {"type": "object", "properties": {
        "query": {"type": "string"},
        "modality": {"type": "string", "enum": ["pdf", "transcript", "log"]}},
        "required": ["query"]},
}


def search_documents(query, modality=None):
    return "\n\n".join(f"SOURCE: {cite(h)}\n{h['content']}" for h in search_chunks(query, modality, 6))


SYSTEM = (DOC_RULE + " Answer questions using two tools: search_documents for what documents, calls "
          "and logs say, and run_pipeline for counts, totals and other numbers in the database. "
          "Cite every claim with its SOURCE (file plus page, time or line) or the collection you "
          "queried. Every number must come from a tool result.\n\n" + describe_mongo())

while True:
    q = input("\nquestion> ").strip()
    if q.lower() in ("quit", "exit"):
        break
    history = [{"role": "user", "content": q}]
    resp, _ = run_with_tools(history, [SEARCH_TOOL, PIPELINE_TOOL],
                             {"search_documents": search_documents, "run_pipeline": run_pipeline},
                             system=SYSTEM, model=HAIKU, module="m9")
    print("\n" + text_of(resp))
```

```bash
python m09_assistant.py
```

Try one document question ("What risks were mentioned on the call?") and one number question ("How many ERROR events per service?").

**Check:** the first uses `search_documents` and cites files; the second uses `run_pipeline` and cites a collection.

## Step 9 — Run 10 test questions

Write 10 questions in `data/m9/questions.txt` (`mkdir -p data/m9`): at least three each for PDFs, transcripts and logs, and one that needs two of them. Run each in `m09_assistant.py` and paste the answers into `notes/m09_results.md`.

**Check:** 10 answers, each with at least one citation.

## Step 10 — Open every citation

For each answer, open the cited source — the PDF page, the recording at that timestamp, the log line (`sed -n '<line>p' data/m7/sample.log`) — and confirm it says what the answer claims. Mark each answer ✅ or ❌ in `notes/m09_results.md`.

**Check:** all 10 are ✅. For any ❌, see whether retrieval missed the right chunk (try a different `k` or route) or Claude misread it.

## Step 11 — See when retrieval doesn't help

Take one small PDF and ask the same question two ways:

```bash
python -c "
from pathlib import Path
from claude_multimodal import ask_pdf, answer_with_sources, text_of
p = sorted(Path('data/m4/pdfs').glob('*.pdf'))[0]
q = 'What are the payment terms?'
print('WHOLE PDF:', text_of(ask_pdf(p, q)))
print('RETRIEVAL:', answer_with_sources(q, modality='pdf'))"
```

**Check:** note which answer is better and which cost more (`python m01_costs.py` style query on module `m4` vs `m9`). Small documents usually answer better whole; retrieval wins when the material is too big for one prompt.

## Step 12 — Commit

```bash
git add claude_multimodal.py m09_*.py notes/m09_results.md .env.example
git commit -m "Module 9: chunks, search index, cited answers, assistant"
git push
```

**Check:** pushed; `.env` still not in git.

## Done when

- [ ] Step 10: on 10 test questions, every answer cites a source you opened and confirmed.
- [ ] Step 8: the assistant chooses between documents and database correctly.
- [ ] Step 11: you can say when retrieval beats sending the whole document.

**Next:** [Module 10 — Production and security](../module-10-production-and-security/README.md)
