# Module 1 — Claude API fundamentals (2–3 days)

[← Module 0](../module-00-setup/README.md) · [Syllabus](../README.md) · [Next: Module 2 →](../module-02-text-analysis/README.md)

**You start with:** Module 0 done — `.env`, MongoDB running, both users working.
**You finish with:** `claude_multimodal.py` containing `ask()`, `text_of()`, `cost_of()`, `log_call()` and a tool loop; a command-line chat (`m01_chat.py`) with a calculator tool; and every API call logged with its cost in MongoDB.

## Key ideas (read once)

- **One call does everything.** Every module uses the Messages API: you send a list of messages and get a reply.
- **No memory.** Claude remembers nothing between calls. A "conversation" is a list *you* keep and resend each time, so longer chats cost more per turn.
- **The reply is a list of blocks.** Usually one `text` block; with tools, also `tool_use` blocks. `stop_reason` says why it stopped: `end_turn` (finished), `max_tokens` (cut off — a bug to fix), `tool_use` (Claude wants your code to run a function).
- **Tools.** You describe a function with a JSON schema. Claude asks for it with a `tool_use` block; your code runs it and sends back a `tool_result`. Every later module uses this.

> Start of session: `cd DataAnalysis_with_LLM && source .venv/bin/activate && docker compose up -d`

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

HAIKU = "claude-haiku-4-5-20251001"
SONNET = "claude-sonnet-5-5"

# USD per million tokens: (input, output). Check Anthropic's pricing page and fill in.
PRICES = {
    HAIKU: (1.00, 5.00),
    SONNET: (None, None),
}

client = anthropic.Anthropic(max_retries=4, timeout=120.0)
db_rw = MongoClient(os.environ["MONGODB_URI_RW"]).course   # scripts write with this
db_ro = MongoClient(os.environ["MONGODB_URI"]).course      # queries read with this
```

- `max_retries=4`: the SDK already retries rate-limit (429), overload (529) and server errors with backoff. Don't write your own retry loop on top.
- Open Anthropic's pricing page now and fill in both rows of `PRICES` (Sonnet's are blank on purpose).

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
    model=HAIKU, max_tokens=600,
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
                temperature=t, max_tokens=20, module="m1")
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
                   max_tokens=2048, max_rounds=10):
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

## Done when

- [ ] Step 10: the chat calls the calculator when needed and prints a running cost.
- [ ] Step 11: the increase in `llm_calls` cost matches the chat's session cost.
- [ ] Step 7: you can explain why the same prompt gives different answers and name the levers that make it more consistent.

**Next:** [Module 2 — Text analysis](../module-02-text-analysis/README.md)
