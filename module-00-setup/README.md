# Module 0 — Setup (1 day)

[Syllabus](../README.md) · [Next: Module 1 →](../module-01-claude-api-fundamentals/README.md)

**You start with:** a GitHub account, a computer with git, Python 3.10+ and Docker.
**You finish with:** a Python environment, secrets in `.env`, a tested Claude API key, a local MongoDB with a read-write user and a read-only user, and a notebook that proves all of it works.

**What this lab is about.** Before you can analyse anything with Claude, you need a working toolbox on your own machine. In this module you make your own copy of the course repo, install Python packages, get an API key, and start a local MongoDB database with two users: one that can write and one that can only read. Every later module builds on this setup, so you finish by proving each piece works from Python.

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

**What you're doing:** making your own copy of the course repo on GitHub and downloading it to your computer. You'll save and push your work there throughout the course, so it has to be a copy you own.

Do this before anything else. You will commit your own code throughout the course, so you need your own copy of the repo, not the original.

1. Sign in to GitHub and open https://github.com/draju1980/DataAnalysis_with_LLM.
2. Click **Fork** (top right), keep your account as the owner, and click **Create fork**.
3. Clone **your fork** (replace `<your-username>` with your GitHub username):

```bash
git clone https://github.com/<your-username>/DataAnalysis_with_LLM.git
```

**Check:** ends with `Resolving deltas: 100%` (or `done.`), and a `DataAnalysis_with_LLM` folder now exists.

4. Go into the new folder:

```bash
cd DataAnalysis_with_LLM
```

Do not clone `draju1980/DataAnalysis_with_LLM` directly: you can't push to it, and your work would have nowhere to go.

Run the check:

```bash
git remote -v
```

**Check:** shows `github.com/<your-username>/DataAnalysis_with_LLM` for `origin` (two lines, `fetch` and `push`).

## Step 1 — Open the project folder and set up `.gitignore`

**What you're doing:** telling git which files must never be uploaded to GitHub: your secrets, the Python environment, and your data. Doing this first means a later `git add` can't leak them by accident.

All work happens in this repo's folder. Go there if you aren't already:

```bash
cd DataAnalysis_with_LLM
```

Write the `.gitignore` file (the whole block is one command):

```bash
cat > .gitignore <<'EOF'
.env
.venv/
.ipynb_checkpoints/
__pycache__/
data/
backups/
EOF
```

- `.env` will hold your secrets (Step 7).
- `data/` will hold your own input files (invoices, recordings…), which may be private.

**Check:**

```bash
cat .gitignore
```

Output shows the six lines above, from `.env` to `backups/`.

Stage the file:

```bash
git add .gitignore
```

Commit it:

```bash
git commit -m "Ignore secrets, virtualenv, data and backups"
```

**Check:** prints `1 file changed, 6 insertions(+)` and `create mode 100644 .gitignore`. On a rerun, `nothing to commit` is fine.

## Step 2 — Create the Python environment

**What you're doing:** creating a private Python installation for this course (a *virtual environment*) and installing the packages it needs: the Claude SDK, a `.env` reader, the MongoDB driver and Jupyter. Keeping them separate means they can't clash with other Python projects on your machine.

A virtual environment keeps this course's packages separate from the rest of your system. Create it:

```bash
python3 -m venv .venv
```

Activate it:

```bash
source .venv/bin/activate
```

**Check:** your prompt now starts with `(.venv)`.

Update pip, the package installer:

```bash
pip install --upgrade pip
```

Install the course packages:

```bash
pip install anthropic python-dotenv pymongo jupyter
```

**Check:** the last line starts with `Successfully installed` (or every package says `Requirement already satisfied`).

Run `source .venv/bin/activate` in every new terminal (see "Start of every session" in the syllabus). Later modules install more packages when they need them.

**Check:**

```bash
python -c "print('\n'); import anthropic, dotenv, pymongo; print('ok')"
```

Output prints:

```
ok
```

## Step 3 — Install Docker and mongosh

**What you're doing:** installing the two tools that run and talk to the database. Docker runs MongoDB in a container so you don't install it directly on your system, and mongosh lets you type commands to MongoDB, which you need to create the database users.

- **Docker** runs MongoDB on your machine. Install Docker Desktop (macOS/Windows) or Docker Engine (Linux).
- **mongosh** is MongoDB's command-line shell, used in Steps 10–12. On Linux, follow "Install mongosh" in the MongoDB docs. On macOS:

  ```bash
  brew install mongosh
  ```

Check each tool, one command at a time. Docker:

```bash
docker --version
```

**Check:** prints `Docker version` and a number.

Docker Compose, which starts MongoDB from a config file in Step 9:

```bash
docker compose version
```

**Check:** prints `Docker Compose version` and a number.

mongosh:

```bash
mongosh --version
```

**Check:** prints a version number such as `2.5.0`.

## Step 4 — Create an Anthropic API key

**What you're doing:** getting the key that lets your code call Claude and pay for it. Every API request carries this key, so treat it like a password.

1. Sign in at console.anthropic.com (separate from the Claude chat app).
2. Add prepaid credit; $10–20 covers Modules 0–4 on Haiku.
3. Create an API key. It starts with `sk-ant-` and is shown only once. Paste it somewhere temporary (a password manager) — you'll move it into `.env` in Step 7.

**Check:** the key shows as active in the Console.

## Step 5 — Set a spend limit

**What you're doing:** capping how much the API can charge you in a month. It's a safety net: a bug can't cost more than the limit you set.

In the Console's billing/limits settings, set a monthly limit slightly above your credit. If a bug in a loop fires thousands of requests, the API refuses them instead of draining your balance.

**Check:** the limit is visible in the Console.

## Step 6 — Generate three database passwords

**What you're doing:** creating three random passwords: one for the MongoDB admin and one each for the read-write and read-only users. Random hex passwords are strong and contain no characters that could break a connection string.

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

**What you're doing:** putting the API key and database connection details in one private file that your code reads at start-up. Secrets stay out of your code and out of git, and you make an empty copy (`.env.example`) that is safe to commit.

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

Now protect the file so only your user account can read it:

```bash
chmod 600 .env
```

Make a template with the same names but no values, which you can commit:

```bash
sed 's/=.*/=/' .env > .env.example
```

Look at the template:

```bash
cat .env.example
```

**Check:** shows the four names, each with nothing after `=`:

```
ANTHROPIC_API_KEY=
MONGO_ROOT_PASSWORD=
MONGODB_URI_RW=
MONGODB_URI=
```

Finally, delete the temporary copy of your API key from Step 4.

Confirm git will never commit `.env`:

```bash
git check-ignore .env
```

**Check:** prints `.env`.

## Step 8 — Test the API key with curl

**What you're doing:** sending one tiny message to Claude with the simplest possible tool, to prove your key and credit work before any Python is involved.

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

**What you're doing:** describing the database container in a small config file and starting it. You get MongoDB plus its search engine running on your own machine, reachable only from your machine, with data that survives restarts.

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

**Check:** the first time, Docker downloads the image (this takes a few minutes), then prints a line ending in `mongodb-1  Started`.

What this does:

- Runs MongoDB **and** its search engine (`mongot`, needed for full-text search in Module 9) in one container.
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

**What you're doing:** logging in to MongoDB as the admin with password 1, to prove the database is running and the password from `.env` reached it. You'll need this login in the next step to create the users.

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

**What you're doing:** creating the two database users your code will log in as: `course_rw`, which can read and write, and `course_ro`, which can only read. Keeping a read-only user is what later makes it safe to run queries that Claude writes.

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

**Check:** the output ends with `users created`. On a rerun, `dropped course_rw` and `dropped course_ro` come first; that's expected.

## Step 12 — Check both users

**What you're doing:** logging in with each connection string from `.env` and asking MongoDB what that user is allowed to do. This proves both passwords work and that the read-only user really is read-only.

Log in as the read-only user (`MONGODB_URI`):

```bash
mongosh "$(grep '^MONGODB_URI=' .env | cut -d= -f2-)" --quiet --eval "db.runCommand({connectionStatus: 1}).authInfo.authenticatedUserRoles"
```

**Check:** prints `[ { role: 'read', db: 'course' } ]`.

Log in as the read-write user (`MONGODB_URI_RW`):

```bash
mongosh "$(grep '^MONGODB_URI_RW=' .env | cut -d= -f2-)" --quiet --eval "db.runCommand({connectionStatus: 1}).authInfo.authenticatedUserRoles"
```

**Check:** prints `[ { role: 'readWrite', db: 'course' } ]`.

If either says "Authentication failed": open `.env`, look for spaces or quotes around values, fix them, and repeat Step 11.

## Step 13 — Prove everything from Python in a notebook

**What you're doing:** checking from Python, the way every later module works, that the pieces fit together: the key in `.env` reaches Claude, the read-write user can save a document, the read-only user can read it, and the read-only user is blocked from writing. The notebook you save is your proof that setup is complete.

Make a folder for notebooks:

```bash
mkdir -p notebooks
```

Start Jupyter:

```bash
jupyter notebook
```

**Check:** a browser tab opens showing the project folder. Jupyter keeps running in this terminal until you stop it: when you've saved the notebook, press Ctrl+C twice in the terminal (or open a second terminal for Step 14).

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
- Cell 3 prints `InsertOneResult(ObjectId('…'), acknowledged=True)`.
- Cell 4 prints `['hello']` and `[{'msg': 'first document'}]`. If you ran Cell 3 more than once, the list has one entry per run; that's fine.
- Cell 5 prints `blocked` twice.

## Step 14 — Commit your setup files

**What you're doing:** saving your setup files to your fork on GitHub, after checking that no secret is in them. From here on, every module ends with a commit like this one.

Commit the files that are safe to share. Never `.env`. Run these one at a time.

See what git would pick up:

```bash
git status --short
```

**Check:** lists `?? .env.example`, `?? docker-compose.yml` and `?? notebooks/`. `.env` must NOT be listed.

Search the three files you are about to commit for an API key or a connection string with a password. Notebook outputs are saved inside the file, so this also catches a cell that printed a secret:

```bash
grep -n "sk-ant-\|mongodb://[^ ]*:[^ ]*@" docker-compose.yml .env.example notebooks/m00_setup.ipynb || echo "no secrets found"
```

**Check:** prints `no secrets found`. If it prints a line instead, remove that secret (for a notebook, clear the cell's output and save) and run it again.

Stage the files:

```bash
git add docker-compose.yml .env.example notebooks/m00_setup.ipynb
```

Commit them:

```bash
git commit -m "Module 0: local setup"
```

**Check:** prints `3 files changed`.

Push to your fork:

```bash
git push
```

**Check:** ends with a line like `main -> main` (your default branch name) and no error.

## Done when

- [ ] Step 8: curl returns a reply with a `usage` block.
- [ ] Step 12: `course_ro` has `read` and `course_rw` has `readWrite` on `course`.
- [ ] Step 13: Cells 1–4 work and Cell 5 prints `blocked` twice.
- [ ] Step 14: no secrets found; setup files pushed; `.env` not in git.

**Next:** [Module 1 — Claude API fundamentals](../module-01-claude-api-fundamentals/README.md)
