# Module 5 — Audio and video (4–5 days)

[← Module 4](../module-04-pdfs-and-documents/README.md) · [Syllabus](../README.md) · [Next: Module 6 →](../module-06-tabular-files/README.md)

**You start with:** Module 4 done — `ask()`, `tool_input()`, `ask_image()` and the extract → store → check pattern.
**You finish with:** `transcribe()`, `save_segments()`, `label_speakers()` and `analyze_audio()`; one command (`m05_process.py`) that turns a one-hour recording into a summary, action items with owners, and timestamped key moments; and a video example that combines frames with the transcript.

## Key ideas (read once)

- **Claude doesn't take audio.** Audio is two steps: transcribe to text with Whisper (running locally), then analyze the text with Claude. Video adds a third: sample frames and send them as images.
- **Whisper is the one non-Claude model in the course.** It only turns speech into text; all the analysis is Claude.
- **Speaker labels come from Claude**, inferred from the text (turn-taking, names, roles). It's less accurate on overlapping speech, so spot-check it.
- **Keep timestamps everywhere** so every answer can point to the moment in the recording.
- **Long recordings:** analyze 10-minute chunks, then summarize the summaries.

> Start of session: `cd DataAnalysis_with_LLM && source .venv/bin/activate && docker compose up -d`

---

## Step 1 — Install faster-whisper and ffmpeg, add a recording

```bash
pip install faster-whisper
brew install ffmpeg                 # macOS; Ubuntu/Debian: sudo apt install ffmpeg
mkdir -p data/m5
```

Put a one-hour recording (earnings call, meeting; mp3, m4a, wav or mp4) in `data/m5/`, e.g. `data/m5/q3-call.mp3`.

**Check:** `ffmpeg -version | head -1` prints a version, and `ls data/m5` shows your recording.

## Step 2 — Make a 3-minute test clip

Build and test every step on a short clip first; run the full hour only at Step 8.

```bash
ffmpeg -y -loglevel error -i data/m5/q3-call.mp3 -t 180 -ac 1 -ar 16000 data/m5/test-clip.wav
```

- `-t 180` keeps the first 180 seconds.
- `-ac 1 -ar 16000` converts to mono 16 kHz, the format Whisper uses.

**Check:** `ffprobe -v error -show_entries format=duration -of csv=p=0 data/m5/test-clip.wav` prints about `180`.

## Step 3 — Add `transcribe()` and transcribe the clip

Append to `claude_multimodal.py`:

```python
def fmt_ts(seconds):
    """1234.5 -> '00:20:34'"""
    s = int(seconds)
    return f"{s // 3600:02d}:{s % 3600 // 60:02d}:{s % 60:02d}"


def transcribe(path, model_size="small"):
    """Transcribe audio locally. Returns a list of {start_s, end_s, text}."""
    from faster_whisper import WhisperModel
    model = WhisperModel(model_size, device="cpu", compute_type="int8")
    segments, _info = model.transcribe(str(path), vad_filter=True)
    return [{"start_s": round(s.start, 1), "end_s": round(s.end, 1), "text": s.text.strip()}
            for s in segments]
```

- `model_size`: `tiny` / `base` are fast but less accurate; `small` is a good start; `medium` is slower and more accurate.
- The first run downloads the Whisper model (a few hundred MB).

**Check:**

```bash
python -c "
from claude_multimodal import transcribe, fmt_ts
segs = transcribe('data/m5/test-clip.wav')
print(len(segs), 'segments'); [print(fmt_ts(s['start_s']), s['text']) for s in segs[:5]]"
```

Prints a segment count and the first lines with timestamps, matching what you hear.

## Step 4 — Add `save_segments()` and store the clip's transcript

Append:

```python
def save_segments(recording, segments):
    """Replace a recording's transcript in MongoDB. Each segment gets an index i."""
    db_rw.transcript_segments.delete_many({"recording": recording})
    db_rw.transcript_segments.insert_many(
        [{"recording": recording, "i": i, "speaker": None, **s} for i, s in enumerate(segments)])
    db_rw.transcript_segments.create_index([("recording", 1), ("i", 1)])
```

```bash
python -c "
from claude_multimodal import transcribe, save_segments, db_ro
save_segments('test-clip', transcribe('data/m5/test-clip.wav'))
print(db_ro.transcript_segments.count_documents({'recording': 'test-clip'}))"
```

**Check:** prints the same segment count as Step 3.

## Step 5 — Add `label_speakers()` and label the clip

Append:

```python
SPEAKER_TOOL = {
    "name": "record_speakers",
    "description": "Record who speaks in each numbered segment.",
    "input_schema": {"type": "object", "properties": {"speakers": {"type": "array", "items": {
        "type": "object",
        "properties": {"i": {"type": "integer"}, "speaker": {"type": "string"}},
        "required": ["i", "speaker"]}}}, "required": ["speakers"]},
}


def label_speakers(recording, chunk=150, model=SONNET):
    """Ask Claude who speaks in each segment, 150 segments at a time."""
    segs = list(db_rw.transcript_segments.find({"recording": recording}).sort("i"))
    known = []
    for start in range(0, len(segs), chunk):
        part = segs[start:start + chunk]
        lines = "\n".join(f"{s['i']}: {s['text']}" for s in part)
        prompt = ("Below are numbered segments of a recording transcript. Decide who speaks in each "
                  "segment using turn-taking, names and roles mentioned. Use real names when they are "
                  "said, otherwise 'Speaker 1', 'Speaker 2', and keep names consistent. "
                  f"Speakers identified so far: {', '.join(known) or 'none'}.\n"
                  f"<transcript>\n{lines}\n</transcript>")
        data = tool_input(ask(prompt, model=model, max_tokens=8000, tools=[SPEAKER_TOOL],
                              tool_choice={"type": "tool", "name": "record_speakers"}, module="m5"))
        for x in data["speakers"]:
            db_rw.transcript_segments.update_one({"recording": recording, "i": x["i"]},
                                                 {"$set": {"speaker": x["speaker"]}})
            if x["speaker"] not in known:
                known.append(x["speaker"])
    return known
```

Passing `known` speakers from chunk to chunk keeps names consistent across a long recording.

```bash
python -c "
from claude_multimodal import label_speakers, db_ro, fmt_ts
print(label_speakers('test-clip'))
for s in db_ro.transcript_segments.find({'recording': 'test-clip'}).sort('i').limit(8):
    print(fmt_ts(s['start_s']), s['speaker'], '-', s['text'])"
```

**Check:** prints the list of speakers and the first segments with sensible speaker names. Listen to the clip and note any wrong labels.

## Step 6 — Add `analyze_audio()` and analyze the clip

Append:

```python
NOTES_TOOL = {
    "name": "record_notes",
    "description": "Record notes for one part of a recording.",
    "input_schema": {"type": "object", "properties": {
        "summary": {"type": "string"},
        "action_items": {"type": "array", "items": {"type": "object", "properties": {
            "owner": {"type": "string"}, "item": {"type": "string"},
            "at": {"type": "string", "description": "hh:mm:ss when it was agreed"}},
            "required": ["owner", "item", "at"]}},
        "key_moments": {"type": "array", "items": {"type": "object", "properties": {
            "at": {"type": "string", "description": "hh:mm:ss"}, "what": {"type": "string"}},
            "required": ["at", "what"]}}},
        "required": ["summary", "action_items", "key_moments"]},
}


def analyze_audio(recording, chunk_minutes=10, model=SONNET):
    """Summarize each chunk, then combine. Returns (summary, action_items, key_moments)."""
    segs = list(db_ro.transcript_segments.find({"recording": recording}).sort("i"))
    chunks, current = [], []
    for s in segs:
        if current and s["start_s"] - current[0]["start_s"] > chunk_minutes * 60:
            chunks.append(current)
            current = []
        current.append(s)
    if current:
        chunks.append(current)

    notes = []
    for c in chunks:
        text = "\n".join(f"[{fmt_ts(s['start_s'])}] {s.get('speaker') or '?'}: {s['text']}" for s in c)
        prompt = ("Summarize this part of a recording. List action items with an owner and the time "
                  "they were agreed, and key moments with times. Use the [hh:mm:ss] times shown.\n"
                  f"<transcript>\n{text}\n</transcript>")
        notes.append(tool_input(ask(prompt, model=model, max_tokens=3000, tools=[NOTES_TOOL],
                                    tool_choice={"type": "tool", "name": "record_notes"}, module="m5")))

    items = [{"recording": recording, **a} for n in notes for a in n["action_items"]]
    db_rw.action_items.delete_many({"recording": recording})
    if items:
        db_rw.action_items.insert_many([dict(i) for i in items])
    moments = [m for n in notes for m in n["key_moments"]]
    summary = text_of(ask("Combine these partial summaries of one recording into a single summary "
                          "of at most 300 words:\n\n" + "\n\n".join(n["summary"] for n in notes),
                          model=model, max_tokens=1000, module="m5"))
    return summary, items, moments
```

```bash
python -c "
from claude_multimodal import analyze_audio
summary, items, moments = analyze_audio('test-clip')
print(summary); print(items); print(moments)"
```

**Check:** a short summary, any action items with owner and time, and key moments with times that match the clip.

## Step 7 — Chain everything into one command

Create `m05_process.py`. It runs Steps 2–6 for any recording and writes a Markdown report.

```python
import subprocess, sys
from pathlib import Path
from claude_multimodal import transcribe, save_segments, label_speakers, analyze_audio

src = Path(sys.argv[1])
recording = src.stem
wav = Path("data/m5") / f"{recording}.wav"

print("1/5 converting audio…")
subprocess.run(["ffmpeg", "-y", "-loglevel", "error", "-i", str(src),
                "-ac", "1", "-ar", "16000", str(wav)], check=True)
print("2/5 transcribing (slowest step)…")
save_segments(recording, transcribe(wav))
print("3/5 labelling speakers…")
speakers = label_speakers(recording)
print("4/5 analysing…")
summary, items, moments = analyze_audio(recording)

out = Path("data/m5") / f"{recording}-notes.md"
lines = [f"# {recording}", "", f"Speakers: {', '.join(speakers)}", "", "## Summary", "", summary,
         "", "## Action items", ""]
lines += [f"- [{a['at']}] **{a['owner']}**: {a['item']}" for a in items] or ["- none"]
lines += ["", "## Key moments", ""] + [f"- [{m['at']}] {m['what']}" for m in moments]
out.write_text("\n".join(lines))
print("5/5 wrote", out)
```

Test it end to end on the clip first:

```bash
python m05_process.py data/m5/test-clip.wav
```

**Check:** prints steps 1/5 to 5/5 and writes `data/m5/test-clip-notes.md`.

## Step 8 — Process the full hour in one command

```bash
time python m05_process.py data/m5/q3-call.mp3
```

Transcription of an hour on CPU can take 10–30 minutes with `small`. Let it run.

**Check:** `data/m5/q3-call-notes.md` exists with a summary, action items with owners and times, and key moments. Jump to three of the timestamps in the recording and confirm they're right.

## Step 9 — Query the recording in MongoDB

```bash
python -c "
from claude_multimodal import db_ro, fmt_ts
print('Action items:')
for a in db_ro.action_items.find({'recording': 'q3-call'}, {'_id': 0}): print(' ', a)
print('Mentions of pricing:')
for s in db_ro.transcript_segments.find({'recording': 'q3-call', 'text': {'\$regex': 'pric', '\$options': 'i'}}).sort('start_s'):
    print(' ', fmt_ts(s['start_s']), s['speaker'], '-', s['text'])"
```

**Check:** action items match the notes file, and the pricing mentions have speakers and timestamps.

## Step 10 — Video: combine frames with the transcript

For a video (`data/m5/demo.mp4`), extract one frame every 30 seconds:

```bash
mkdir -p data/m5/frames
ffmpeg -y -loglevel error -i data/m5/demo.mp4 -vf fps=1/30 data/m5/frames/%04d.jpg
python m05_process.py data/m5/demo.mp4          # transcript + notes, as in Step 7
```

Frame `0001.jpg` is at 0:00, `0002.jpg` at 0:30, and so on. Now ask about one 2-minute window using both. Create `m05_video_window.py`:

```python
import sys
from pathlib import Path
from claude_multimodal import ask_image, db_ro, fmt_ts

start, end = int(sys.argv[1]), int(sys.argv[2])          # seconds, e.g. 60 180
frames = [p for p in sorted(Path("data/m5/frames").glob("*.jpg"))
          if start <= (int(p.stem) - 1) * 30 < end]
segs = db_ro.transcript_segments.find({"recording": "demo", "start_s": {"$gte": start, "$lt": end}}).sort("i")
transcript = "\n".join(f"[{fmt_ts(s['start_s'])}] {s.get('speaker') or '?'}: {s['text']}" for s in segs)
print(ask_image(frames, "These frames come from this part of a video, in order. "
                "Using both the frames and the transcript, describe what is shown and said, "
                f"with timestamps.\n<transcript>\n{transcript}\n</transcript>", max_side=1024))
```

```bash
python m05_video_window.py 60 180
```

**Check:** the answer mentions things only visible in the frames (slides, screens) together with what was said.

## Step 11 — Commit

```bash
git add claude_multimodal.py m05_*.py
git commit -m "Module 5: transcription, speaker labels, audio analysis, video frames"
git push
```

**Check:** pushed; no recordings from `data/` in the commit.

## Done when

- [ ] Step 8: `python m05_process.py <recording>` processes the full hour end to end in one command.
- [ ] Step 8: three timestamps checked against the recording.
- [ ] Step 10: a video window answered from frames and transcript together.

**Next:** [Module 6 — Tabular files](../module-06-tabular-files/README.md)
