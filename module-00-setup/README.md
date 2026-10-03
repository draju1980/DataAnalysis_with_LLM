# Module 0 — Setup (1 day)

[Syllabus](../README.md) · [Next: Module 1 →](../module-01-claude-api-fundamentals/README.md)

**You start with:** a GitHub account, a computer with git, Python 3.10+ and Docker.
**You finish with:** a Python environment, secrets in `.env`, a tested Claude API key, a local MongoDB with a read-write user and a read-only user, and a notebook that proves all of it works.

## Key ideas (read once)

- **The API.** Your code sends an HTTPS request with text (later also images and PDFs) and gets Claude's reply back as JSON.
- **Tokens.** Claude reads text in chunks called tokens; 1,000 tokens is roughly 750 English words. You pay per token: input (what you send) and output (what Claude writes) have separate prices, and output costs more.
- **Model.** The whole course uses one model, `claude-haiku-4-5-20251001` (Claude Haiku 4.5), with `max_tokens=512` on every call. `max_tokens` is the longest reply Claude may write.
- **Why a database.** Every later module saves Claude's results into MongoDB so you can check accuracy and cost with one query. The read-only user you create here is what makes it safe to let Claude write queries in Module 8.

## Before you start or resume

Setup can take more than one sitting: installing Docker or waiting for API credit, for example. At the start of every session, open a terminal and go to the project folder:

```bash
cd DataAnalysis_with_LLM
```

Then run this to see which steps are already done. It uses only the shell, so it works before the Python environment exists.

```bash
(
  step() { if eval "$2" >/dev/null 2>&1; then echo "done  $1"; else echo "todo  $1"; fi; }
  step "Step 0   origin is your fork"         'git remote get-url origin | grep -v draju1980'
  step "Step 1   .gitignore protects .env"    'grep -qxF .env .gitignore'
  step "Step 2   Python environment"          '.venv/bin/python -c "import anthropic, dotenv, pymongo"'
  step "Step 3   Docker and mongosh"          'docker compose version && mongosh --version'
  step "Step 7   .env has all four values"    'test $(grep -cE "^(ANTHROPIC_API_KEY|MONGO_ROOT_PASSWORD|MONGODB_URI_RW|MONGODB_URI)=." .env) -eq 4 && test -f .env.example'
  step "Step 9   MongoDB is healthy"          'docker compose ps mongodb | grep -q "(healthy)"'
  step "Step 11  course_rw can log in"        'mongosh "$(grep "^MONGODB_URI_RW=" .env | cut -d= -f2-)&serverSelectionTimeoutMS=3000" --quiet --eval "quit(db.runCommand({connectionStatus: 1}).authInfo.authenticatedUsers.some(u => u.user === \"course_rw\") ? 0 : 1)"'
  step "Step 11  course_ro can log in"        'mongosh "$(grep "^MONGODB_URI=" .env | cut -d= -f2-)&serverSelectionTimeoutMS=3000" --quiet --eval "quit(db.runCommand({connectionStatus: 1}).authInfo.authenticatedUsers.some(u => u.user === \"course_ro\") ? 0 : 1)"'
  step "Step 13  notebook saved"              'test -f notebooks/m00_setup.ipynb'
  step "Step 14  setup committed"             'git log --oneline --author="$(git config user.email)" | grep -q "Module 0:"'
)
```

**Check:** resume at the first `todo` line. Steps 4–6 and 8 happen in the Anthropic Console or print to the screen, so they have no line of their own: Step 7 being `done` means you finished them. If MongoDB was stopped, the Step 9 and 11 lines show `todo` until you start it, wait a minute, and run the block again:

```bash
docker compose up -d
```

**Resuming safely**

- Once Step 2 is done, start every new terminal by activating the Python environment:

  ```bash
  source .venv/bin/activate
  ```

  Once Step 9 is done, also start MongoDB:

  ```bash
  docker compose up -d
  ```
- The API key is shown only once (Step 4). If you lost it before putting it in `.env`, create a new key and delete the old one in the Console.
- Don't generate new passwords (Step 6) after Step 11 unless you also put them in `.env` and rerun Step 11: the users keep the passwords they were created with.
- Steps 9, 11 and 12 are safe to rerun. Step 11 drops and recreates both users.
- **To stop for the day**, stop MongoDB (or leave it running; your data stays):

  ```bash
  docker compose stop
  ```

  Never run `docker compose down -v` to pause: `-v` deletes the database volumes, and you would have to redo Steps 9–12.

---

## Step 0 — Fork the course repo to your GitHub account

Do this before anything else. You will commit your own code throughout the course, so you need your own copy of the repo, not the original.

1. Sign in to GitHub and open https://github.com/draju1980/DataAnalysis_with_LLM.
2. Click **Fork** (top right), keep your account as the owner, and click **Create fork**.
3. Clone **your fork** (replace `<your-username>` with your GitHub username):

```bash
git clone https://github.com/<your-username>/DataAnalysis_with_LLM.git
cd DataAnalysis_with_LLM
```

Do not clone `draju1980/DataAnalysis_with_LLM` directly: you can't push to it, and your work would have nowhere to go.

Run the check:

```bash
git remote -v
```

**Check:** shows `github.com/<your-username>/DataAnalysis_with_LLM` for `origin`.

## Step 1 — Open the project folder and set up `.gitignore`

All work happens in this repo's folder. First tell git which files must never be committed.

```bash
cd DataAnalysis_with_LLM
cat > .gitignore <<'EOF'
.env
.venv/
.ipynb_checkpoints/
__pycache__/
data/
backups/
EOF
git add .gitignore
git commit -m "Ignore secrets, virtualenv, data and backups"
```

- `.env` will hold your secrets (Step 7).
- `data/` will hold your own input files (invoices, recordings…), which may be private.

**Check:**

```bash
cat .gitignore
```

Output shows six lines.

## Step 2 — Create the Python environment

A virtual environment keeps this course's packages separate from the rest of your system.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install anthropic python-dotenv pymongo jupyter
```

Your prompt now starts with `(.venv)`. Run `source .venv/bin/activate` in every new terminal (see "Start of every session" in the syllabus). Later modules install more packages when they need them.

**Check:**

```bash
python -c "print('\n'); import anthropic, dotenv, pymongo; print('ok')"
```

Output prints:

```
ok
```

## Step 3 — Install Docker and mongosh

- **Docker** runs MongoDB on your machine. Install Docker Desktop (macOS/Windows) or Docker Engine (Linux).
- **mongosh** is MongoDB's command-line shell, used in Steps 10–12. On Linux, follow "Install mongosh" in the MongoDB docs. On macOS:

  ```bash
  brew install mongosh
  ```

**Check:**

```bash
docker --version
docker compose version
mongosh --version
```

All three print a version.

## Step 4 — Create an Anthropic API key

1. Sign in at console.anthropic.com (separate from the Claude chat app).
2. Add prepaid credit; $10–20 covers Modules 0–4 on Haiku.
3. Create an API key. It starts with `sk-ant-` and is shown only once. Paste it somewhere temporary (a password manager) — you'll move it into `.env` in Step 7.

**Check:** the key shows as active in the Console.

## Step 5 — Set a spend limit

In the Console's billing/limits settings, set a monthly limit slightly above your credit. If a bug in a loop fires thousands of requests, the API refuses them instead of draining your balance.

**Check:** the limit is visible in the Console.

## Step 6 — Generate three database passwords

```bash
for i in 1 2 3; do openssl rand -hex 24; done
```

You get three 48-character passwords made of `0-9` and `a-f` only (no symbols that could break a connection string). Keep the terminal open; you paste them into `.env` in the next step.

| Password | Used for |
| --- | --- |
| 1 | MongoDB admin account (only for creating users) |
| 2 | `course_rw` — read-write user for your scripts |
| 3 | `course_ro` — read-only user for Claude's queries |

**Check:** three lines of hex printed.

## Step 7 — Create the `.env` file

Create a file named `.env` in the project folder with exactly this content, replacing the placeholders with your key (Step 4) and passwords (Step 6):

```
ANTHROPIC_API_KEY=sk-ant-...
MONGO_ROOT_PASSWORD=<password 1>
MONGODB_URI_RW=mongodb://course_rw:<password 2>@127.0.0.1:27017/course?authSource=admin&directConnection=true
MONGODB_URI=mongodb://course_ro:<password 3>@127.0.0.1:27017/course?authSource=admin&directConnection=true
```

Rules: no `export`, no spaces around `=`, no quotes around values.

- The plain `MONGODB_URI` is the **read-only** user, so the default is the safe one.
- `authSource=admin` tells MongoDB where the users are stored.
- `directConnection=true` is needed because the local MongoDB runs as a one-machine replica set.

Now protect the file and make a template you can commit:

```bash
chmod 600 .env
sed 's/=.*/=/' .env > .env.example
cat .env.example
```

Finally, delete the temporary copy of your API key from Step 4.

Run the check:

```bash
git check-ignore .env
```

**Check:** prints `.env`, and `.env.example` shows the four names with nothing after `=`.

## Step 8 — Test the API key with curl

This sends one tiny request straight to the API using the key in `.env`, with no Python involved. If it works, any later problem is in your code, not your key.

```bash
(
  key=$(grep '^ANTHROPIC_API_KEY=' .env | cut -d= -f2-)
  curl -sS https://api.anthropic.com/v1/messages \
    --header "x-api-key: $key" \
    --header "anthropic-version: 2023-06-01" \
    --header "content-type: application/json" \
    --data '{"model": "claude-haiku-4-5-20251001", "max_tokens": 512,
      "messages": [{"role": "user", "content": "Hello, world"}]}'
)
```

The key is read from `.env`, so it never appears in your shell history; the brackets make the variable disappear afterwards.

| If you see | It means |
| --- | --- |
| `authentication_error` (401) | The key in `.env` is wrong or mis-copied |
| A credit balance error | The account has no prepaid credit (Step 4) |
| `not_found_error` | Typo in the model name |
| Connection or timeout error | Network or proxy problem |

**Check:** the reply is JSON with Claude's text and a `usage` block containing `input_tokens` and `output_tokens`.

## Step 9 — Start MongoDB

Create `docker-compose.yml` in the project folder:

```yaml
services:
  mongodb:
    image: mongodb/mongodb-atlas-local:8.0
    hostname: mongodb
    environment:
      MONGODB_INITDB_ROOT_USERNAME: admin
      MONGODB_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}
    ports:
      - "127.0.0.1:27017:27017"
    volumes:
      - db:/data/db
      - configdb:/data/configdb
      - mongot:/data/mongot
volumes:
  db:
  configdb:
  mongot:
```

Start it:

```bash
docker compose up -d
```

What this does:

- Runs MongoDB **and** its search engine (`mongot`, needed for vector search in Module 9) in one container.
- `8.0` pins the MongoDB version so the setup doesn't change under you.
- `127.0.0.1:27017:27017` makes the database reachable **only from your machine**.
- Compose reads `${MONGO_ROOT_PASSWORD}` from `.env` (Step 7); only that one value goes into the container.
- The three volumes keep your data when the container restarts.

**Check:** wait about a minute, then run:

```bash
docker compose ps
```

Output shows `mongodb` as **healthy**. If it says `starting`, wait and check again.

## Step 10 — Check the admin login

```bash
mongosh "mongodb://admin@127.0.0.1:27017/admin?directConnection=true"
```

When asked for a password, paste **password 1**. You should see a prompt like:

```
AtlasLocalDev mongodb [direct: primary] admin>
```

Type `exit` to leave. The next step runs in your normal terminal, not inside mongosh.

**Check:** you reached the prompt above without "Authentication failed".

## Step 11 — Create the two users

This command reads passwords 2 and 3 from `.env` and creates both users. It first removes either user if it already exists, so it is safe to run again.

```bash
(
  get() { grep "^$1=" .env | cut -d= -f2- | tr -d '\r'; }
  pw()  { get "$1" | sed -E 's#^mongodb://[^:]+:([^@]+)@.*#\1#'; }
  export RW_PW="$(pw MONGODB_URI_RW)" RO_PW="$(pw MONGODB_URI)"
  mongosh "mongodb://admin@127.0.0.1:27017/admin?directConnection=true" --quiet --eval '
    for (const u of ["course_rw", "course_ro"]) {
      if (db.getUser(u)) { db.dropUser(u); print("dropped " + u); }
    }
    db.createUser({user: "course_rw", pwd: process.env.RW_PW,
                   roles: [{role: "readWrite", db: "course"}]});
    db.createUser({user: "course_ro", pwd: process.env.RO_PW,
                   roles: [{role: "read", db: "course"}]});
    print("users created");'
)
```

When asked, paste **password 1** (admin).

How it works:

1. `get` reads one line from `.env` and strips Windows line endings.
2. `pw` cuts the password out of each connection string (between `user:` and `@`).
3. The passwords reach mongosh as environment variables that exist only inside the brackets — never on screen or in a history file.

> **Don't paste passwords into `passwordPrompt()` inside mongosh.** Terminals add hidden characters to pasted text, and MongoDB rejects the password with `U_STRINGPREP_PROHIBITED_ERROR`, however simple or strong it is.

What you now have:

- **course_rw** — can read and write the `course` database. Your scripts use it.
- **course_ro** — can only read the `course` database. No inserts, updates, deletes, `$out` or `$merge`. Claude's queries use it.

**Check:** the output ends with `users created`.

## Step 12 — Check both users

```bash
mongosh "$(grep '^MONGODB_URI=' .env | cut -d= -f2-)" --quiet --eval "db.runCommand({connectionStatus: 1}).authInfo.authenticatedUserRoles"
mongosh "$(grep '^MONGODB_URI_RW=' .env | cut -d= -f2-)" --quiet --eval "db.runCommand({connectionStatus: 1}).authInfo.authenticatedUserRoles"
```

**Check:** the first prints `role: 'read', db: 'course'`; the second prints `role: 'readWrite', db: 'course'`.

If either says "Authentication failed": open `.env`, look for spaces or quotes around values, fix them, and repeat Step 11.

## Step 13 — Prove everything from Python in a notebook

```bash
mkdir -p notebooks
jupyter notebook
```

In the browser, open the `notebooks` folder, create a new notebook named `m00_setup`, and run these cells in order.

**Cell 1 — load secrets from `.env`** (always run first):

```python
from dotenv import load_dotenv
load_dotenv("../.env")
```

**Cell 2 — talk to Claude:**

```python
import anthropic

client = anthropic.Anthropic()          # reads ANTHROPIC_API_KEY
resp = client.messages.create(
    model="claude-haiku-4-5-20251001",
    max_tokens=512,
    messages=[{"role": "user", "content": "hello"}],
)
print(resp.content[0].text)
print(resp.usage)
```

`resp.usage` shows the tokens you paid for. Multiply by the per-million prices on Anthropic's pricing page once by hand to see what this call cost.

**Cell 3 — write as `course_rw`:**

```python
import os
from pymongo import MongoClient

rw = MongoClient(os.environ["MONGODB_URI_RW"]).course
rw.hello.insert_one({"msg": "first document"})
```

A MongoDB database holds **collections** (like tables), and each collection holds **documents** (JSON-like records). Both are created automatically on first insert.

**Cell 4 — read as `course_ro`:**

```python
ro = MongoClient(os.environ["MONGODB_URI"]).course
print(ro.list_collection_names())
print(list(ro.hello.find({}, {"_id": 0})))
```

**Cell 5 — prove the read-only user cannot write:**

```python
from pymongo.errors import OperationFailure

attempts = {
    "insert": lambda: ro.hello.insert_one({"msg": "nope"}),
    "$out stage": lambda: list(ro.hello.aggregate([{"$out": "copy"}])),
}
for name, attempt in attempts.items():
    try:
        attempt()
        print(name, "SUCCEEDED — check the user's roles!")
    except OperationFailure as e:
        print(name, "blocked:", e.details.get("errmsg"))
```

`$out` writes query results into a collection, so it's a write hidden inside a query. Module 8 relies on the read-only user blocking it.

**Check:**

- Cell 2 prints a greeting and a `Usage(...)` line.
- Cell 4 prints `['hello']` and `[{'msg': 'first document'}]`.
- Cell 5 prints `blocked` twice.

## Step 14 — Commit your setup files

Commit the files that are safe to share. Never `.env`.

Run these one at a time. In the `git status` output, `.env` must NOT be listed:

```bash
git status --short
grep -rn "sk-ant-\|mongodb://[^ ]*:[^ ]*@" --exclude=.env --exclude-dir=.venv . || echo "no secrets found"
git add docker-compose.yml .env.example notebooks/m00_setup.ipynb
git commit -m "Module 0: local setup"
git push
```

Before committing the notebook, make sure no cell printed a key or connection string; notebook outputs are saved inside the file.

**Check:** the grep prints `no secrets found`, and `git push` succeeds.

## Done when

- [ ] Step 8: curl returns a reply with a `usage` block.
- [ ] Step 12: `course_ro` has `read` and `course_rw` has `readWrite` on `course`.
- [ ] Step 13: Cells 1–4 work and Cell 5 prints `blocked` twice.
- [ ] Step 14: no secrets found; setup files pushed; `.env` not in git.

**Next:** [Module 1 — Claude API fundamentals](../module-01-claude-api-fundamentals/README.md)
