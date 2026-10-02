# Module 1 — Claude API fundamentals (2–3 days)

[← Module 0](../module-00-setup/README.md) · [Syllabus](../README.md) · [Next: Module 2 →](../module-02-text-analysis/README.md)

**You start with:** Module 0 done — `.env`, MongoDB running, both users working.
**You finish with:** `claude_multimodal.py` containing `ask()`, `text_of()`, `cost_of()`, `log_call()` and a tool loop; a command-line chat (`m01_chat.py`) with a calculator tool; and every API call logged with its cost in MongoDB.

## Key ideas (read once)

- **One call does everything.** Every module uses the Messages API: you send a list of messages and get a reply.
- **No memory.** Claude remembers nothing between calls. A "conversation" is a list *you* keep and resend each time, so longer chats cost more per turn.
- **The reply is a list of blocks.** Usually one `text` block; with tools, also `tool_use` blocks. `stop_reason` says why it stopped: `end_turn` (finished), `max_tokens` (cut off — a bug to fix), `tool_use` (Claude wants your code to run a function).
- **Tools.** You describe a function with a JSON schema. Claude asks for it with a `tool_use` block; your code runs it and sends back a `tool_result`. Every later module uses this.

## Before you start or resume

You probably won't finish this module in one sitting. Run these three blocks at the start of **every** session.

**1. Start the session**

```bash
cd DataAnalysis_with_LLM
source .venv/bin/activate
docker compose up -d
until docker compose ps mongodb | grep -q "(healthy)"; do sleep 3; done; echo "MongoDB ready"
```

**2. Check the prerequisites** (Module 0)

```bash
python -c "
import os
from dotenv import load_dotenv
from pymongo import MongoClient
load_dotenv()
for name in ('MONGODB_URI_RW', 'MONGODB_URI'):
    MongoClient(os.environ[name], serverSelectionTimeoutMS=5000).course.command('ping')
print('Module 0: .env and both database users ok')"
```

**Check:** prints `Module 0: … ok`. If not, run the "Before you start or resume" check in [Module 0](../module-00-setup/README.md#before-you-start-or-resume).

**3. Find where you stopped**

```bash
(
  step() { if eval "$2" >/dev/null 2>&1; then echo "done  $1"; else echo "todo  $1"; fi; }
  step "Step 1   claude_multimodal.py imports"  'python -c "import claude_multimodal"'
  step "Step 2   text_of(), cost_of()"          'grep -qF "def cost_of(" claude_multimodal.py'
  step "Step 3   log_call()"                    'grep -qF "def log_call(" claude_multimodal.py'
  step "Step 4   ask()"                         'grep -qF "def ask(" claude_multimodal.py'
  step "Step 6   m01_stream.py"                 'test -f m01_stream.py'
  step "Step 7   m01_temperature.py"            'test -f m01_temperature.py'
  step "Step 8   calc() and CALC_TOOL"          'grep -qF "def calc(" claude_multimodal.py'
  step "Step 9   run_with_tools()"              'grep -qF "def run_with_tools(" claude_multimodal.py'
  step "Step 10  m01_chat.py"                   'test -f m01_chat.py'
  step "Step 11  m01_costs.py"                  'test -f m01_costs.py'
  step "Step 12  committed"                     'git log --oneline --author="$(git config user.email)" | grep -q "Module 1:"'
)
```

**Check:** resume at the first `todo` line. Step 5 only reads the log, so it has no line.

**Resuming safely**

- Never paste an "Append to `claude_multimodal.py`" block a second time: a function defined twice silently uses the last copy. If a step's line says `done`, skip its append.
- If the Step 1 line says `todo` but the file exists, the file has an error, often a block pasted halfway before a break. Run `python -c "import claude_multimodal"` to see the line, and fix it in place.
- Steps 4–7 and 10 call the API again when rerun. Each call costs a fraction of a cent and adds a row to `llm_calls`, which is fine.
- Step 11 compares the cost before and after one chat session. Do the "before", the chat, and the "after" in the same sitting.
- Not sure your `claude_multimodal.py` is right after a break? Compare it with the [complete file for this module](#complete-claude_multimodalpy-after-module-1) at the end of the page.
- **To stop for the day**, run `docker compose stop` or leave MongoDB running. Never `docker compose down -v`: it deletes the database.

---

## Step 1 — Create `claude_multimodal.py` with the shared setup

Create `claude_multimodal.py` in the project root. Every function in the course goes into this file.

```python
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
```

- `max_retries=4`: the SDK already retries rate-limit (429), overload (529) and server errors with backoff. Don't write your own retry loop on top.
- Every call in the course uses `model="claude-haiku-4-5-20251001"` and `max_tokens=1024`. Check Haiku's price on Anthropic's pricing page and correct `PRICES` if it has changed.

**Check:** `python -c "import claude_multimodal; print('ok')"` prints `ok`.

## Step 2 — Add `text_of()` and `cost_of()`

Append to `claude_multimodal.py`:

```python
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
```

**Check:** `python -c "from claude_multimodal import text_of, cost_of; print('ok')"` prints `ok`. (You'll test them with a real reply in Step 4.)

## Step 3 — Add `log_call()` to record every call in MongoDB

Append:

```python
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
```

From now on, every cost question in the course is a query on `llm_calls`.

**Check:** `python -c "from claude_multimodal import log_call; print('ok')"` prints `ok`.

## Step 4 — Add `ask()`, the one function every module calls

Append:

```python
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
```

- `prompt` is for one-off questions; `messages` is for conversations (Step 10).
- `module` tags the call in `llm_calls`, so you can see cost per module later.

**Check:**

```bash
python -c "
from claude_multimodal import ask, text_of, cost_of
r = ask('In one sentence, what is a Docker volume?', module='m1')
print(text_of(r)); print(r.usage); print('cost USD', cost_of(r))"
```

Prints one sentence, a usage line and a small cost (a fraction of a cent).

## Step 5 — Look at the log in MongoDB

The call from Step 4 is now a document in `llm_calls`. Read it with the read-only connection:

```bash
python -c "
from claude_multimodal import db_ro
for d in db_ro.llm_calls.find({}, {'_id': 0}).sort('ts', -1).limit(3): print(d)"
```

**Check:** you see your Step 4 call with `module: 'm1'`, token counts, `cost_usd` and `latency_ms`.

## Step 6 — Stream a long reply

For long answers, streaming prints text as it's generated. Create `m01_stream.py`:

```python
from claude_multimodal import client, log_call, HAIKU

with client.messages.stream(
    model=HAIKU, max_tokens=1024,
    messages=[{"role": "user", "content": "Explain how Kubernetes liveness and readiness probes differ, in 6 sentences."}],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
    final = stream.get_final_message()

log_call(final, "m1", 0)
print("\n\n", final.usage, final.stop_reason)
```

```bash
python m01_stream.py
```

**Check:** text appears gradually, then the usage line and `end_turn`.

## Step 7 — See why the same prompt gives different answers

Claude picks each word by sampling; `temperature` controls how random that is. Create `m01_temperature.py`:

```python
from claude_multimodal import ask, text_of

for t in (1.0, 0.0):
    print(f"--- temperature {t}")
    for _ in range(4):
        r = ask("Suggest one unusual name for a cat. Reply with the name only.",
                temperature=t, max_tokens=1024, module="m1")
        print(text_of(r))
```

```bash
python m01_temperature.py
```

**Check:** at 1.0 the names vary; at 0.0 they're the same or nearly the same. Temperature 0 is *more consistent*, not guaranteed identical. The other levers for consistency come later: tighter instructions and examples (Module 2), forced structured output (Module 2), and pinning a dated model version (you already do: `claude-haiku-4-5-20251001`).

## Step 8 — Add a safe calculator tool

Claude is not reliable at arithmetic, so give it a calculator. The calculator must never run arbitrary code: tool inputs are written by the model, and in later modules the model reads untrusted files. So it parses the expression and allows only numbers and arithmetic operators — never `eval()`.

Append to `claude_multimodal.py`:

```python
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
```

**Check:**

```bash
python -c "
from claude_multimodal import calc
print(calc('1,847 * 0.05'))
try: calc('__import__(\"os\").system(\"ls\")')
except ValueError as e: print('blocked:', e)"
```

Prints `92.35` and `blocked: only numbers and + - * / % ** are allowed`.

## Step 9 — Add the tool loop

When Claude wants a tool, your code must run it and send the result back, repeating until Claude gives a final answer. Append:

```python
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
```

- `history` is the conversation list; the function appends Claude's replies and the tool results to it.
- `handlers` maps a tool name to the Python function that runs it, e.g. `{"calculator": calc}`.

**Check:** `python -c "from claude_multimodal import run_with_tools, tool_input; print('ok')"` prints `ok`.

## Step 10 — Build the command-line chat

Create `m01_chat.py`:

```python
from claude_multimodal import run_with_tools, text_of, cost_of, calc, CALC_TOOL

history = []      # the whole conversation, resent on every turn
total = 0.0       # running cost in USD

print("Chat with Claude. Type 'quit' to exit.")
while True:
    user = input("\nyou> ").strip()
    if user.lower() in ("quit", "exit"):
        break
    if not user:
        continue
    history.append({"role": "user", "content": user})
    resp, replies = run_with_tools(history, [CALC_TOOL], {"calculator": calc}, module="m1")
    total += sum(cost_of(r) or 0 for r in replies)
    print(f"\nclaude> {text_of(resp)}")
    print(f"  [{len(replies)} call(s) | input tokens this turn: {resp.usage.input_tokens} "
          f"| running cost: ${total:.5f}]")
print(f"Session cost: ${total:.5f}")
```

Run it and type these three messages in order:

```bash
python m01_chat.py
```

1. `What's 18% VAT on 1,847 AED?` → you should see `[tool] calculator …` before the answer.
2. `What is the capital of Japan?` → no tool call.
3. `Add 5% to the VAT amount from before.` → works only because `history` carries the earlier answer.

Then type `quit` and note the **Session cost**.

**Check:** the tool is called for 1 and 3 but not 2, and input tokens grow on every turn (the whole history is resent).

## Step 11 — Confirm the cost from MongoDB

Your chat's running total must match what `llm_calls` recorded. Create `m01_costs.py`:

```python
from claude_multimodal import db_ro

pipeline = [
    {"$match": {"module": "m1"}},                      # like SQL WHERE
    {"$group": {"_id": "$model",                        # like SQL GROUP BY
                "calls": {"$sum": 1},
                "cost_usd": {"$sum": "$cost_usd"},
                "avg_ms": {"$avg": "$latency_ms"}}},
]
for row in db_ro.llm_calls.aggregate(pipeline):
    print(row)
```

```bash
python m01_costs.py
```

An **aggregation pipeline** is a list of stages; each stage transforms the documents and passes them on. You'll use them in every module.

**Check:** the total includes your chat session (plus Steps 4–7). To compare exactly, note the cost before and after one more short chat session — the increase must equal the chat's **Session cost**.

## Step 12 — Commit your work

```bash
git add claude_multimodal.py m01_stream.py m01_temperature.py m01_chat.py m01_costs.py
git commit -m "Module 1: ask(), logging, tool loop, CLI chat"
git push
```

**Check:** `git status --short` shows nothing left to commit except files you chose not to add.

## Complete `claude_multimodal.py` after Module 1

Use this to cross-check your file once the steps are done, or after a break. It is every block the course has told you to add to `claude_multimodal.py` through Module 1, in order. The `# ── Module N, Step M ──` lines only show which step added the code below them; your file doesn't need them.

To compare automatically, save the file below as `data/expected.py` (`data/` is git-ignored, so it never gets committed), then:

```bash
diff -Bw <(grep -v '^# ── ' data/expected.py) claude_multimodal.py && echo "your file matches"
```

`-Bw` ignores blank lines and spacing. Every other line `diff` prints is a real difference: a missing step, a block pasted twice, or a typo.

<details>
<summary>Show the complete file (155 lines)</summary>

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
```

</details>


## Done when

- [ ] Step 10: the chat calls the calculator when needed and prints a running cost.
- [ ] Step 11: the increase in `llm_calls` cost matches the chat's session cost.
- [ ] Step 7: you can explain why the same prompt gives different answers and name the levers that make it more consistent.

**Next:** [Module 2 — Text analysis](../module-02-text-analysis/README.md)
