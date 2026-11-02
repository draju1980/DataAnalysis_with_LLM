# Module 0 — Setup (1 day)

[Syllabus](../README.md) · [Next: Module 1 →](../module-01-claude-api-fundamentals/README.md)

**You start with:** a GitHub account, a computer with git, and Docker.
**You finish with:** secrets in `.env`, a tested Claude API key, a local MongoDB (via Docker) with a read-write user and a read-only user, and a notebook that proves all of it works.

**What this lab is about.** Before you can analyse anything with Claude, you need a working toolbox on your own machine. In this module you make your own copy of the course repo, get an API key, and start a local MongoDB database via Docker with two users: one that can write and one that can only read. All Python dependencies are included in the Docker container. Every later module builds on this setup, so you finish by proving each piece works from the container.

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

Then run this to see which steps are already done. It uses only the shell, so it works before Docker containers are running.

```bash
(
  step() { if eval "$2" >/dev/null 2>&1; then echo "done  $1"; else echo "todo  $1"; fi; }
  step "Step 0   origin is your fork"         'git remote get-url origin | grep -v draju1980'
  step "Step 1   Docker"                      'docker compose version'
  step "Step 4   .env has all four values"    'test $(grep -cE "^(ANTHROPIC_API_KEY|MONGO_ROOT_PASSWORD|MONGODB_URI_RW|MONGODB_URI)=." .env) -eq 4 && test -f .env.example'
  step "Step 6   MongoDB is healthy"          'docker compose ps mongodb | grep -q "(healthy)"'
  step "Step 8   course_rw can log in"        'docker compose exec -T mongodb mongosh "$(grep "^MONGODB_URI_RW=" .env | cut -d= -f2-)&serverSelectionTimeoutMS=3000" --quiet --eval "quit(db.runCommand({connectionStatus: 1}).authInfo.authenticatedUsers.some(u => u.user === \"course_rw\") ? 0 : 1)"'
  step "Step 8   course_ro can log in"        'docker compose exec -T mongodb mongosh "$(grep "^MONGODB_URI=" .env | cut -d= -f2-)&serverSelectionTimeoutMS=3000" --quiet --eval "quit(db.runCommand({connectionStatus: 1}).authInfo.authenticatedUsers.some(u => u.user === \"course_ro\") ? 0 : 1)"'
  step "Step 10  notebook saved"              'test -f notebooks/m00_setup.ipynb'
)
```

**Check:** resume at the first `todo` line. Steps 2–3 and 5 happen in the Anthropic Console or only print to the screen, so they have no line of their own: Step 4 being `done` means you finished them. If MongoDB was stopped, the Step 6 and 8 lines show `todo` until you start it, wait a minute, and run the block again:

```bash
docker compose up -d
```

**Resuming safely**

- Once Step 1 is done, start every new terminal session with Docker:

  ```bash
  docker compose up -d
  ```

- The API key is shown only once (Step 2). If you lost it before putting it in `.env`, create a new key and delete the old one in the Console.
- Step 3 never overwrites an existing `.env`, so it's safe to rerun. Don't delete `.env` to get new passwords after Step 8: the database users keep the passwords they were created with.
- Steps 6, 8 and 9 are safe to rerun. Step 8 drops and recreates both users.
- **To stop for the day**, stop the Docker containers (or leave them running; your data stays):

  ```bash
  docker compose stop
  ```

  Never run `docker compose down -v` to pause: `-v` deletes the database volumes, and you would have to redo Steps 6–9.

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

## Step 1 — Open the project folder

**What you're doing:** preparing the course repository to run inside Docker.

All work happens in this repo's folder. Go there if you aren't already:

```bash
cd DataAnalysis_with_LLM
```

The `.gitignore` file is already included in the repo to protect your secrets and data.

**Check:**

```bash
cat .gitignore
```

Output shows that `.env`, `data/`, and other local files are ignored.

## Step 2 — Install Docker

**What you're doing:** installing Docker, which runs all course code in containers. This means all Python packages and dependencies are pre-installed in the Docker image, so you don't install them on your local machine.

Install **Docker Desktop** for your operating system:
- **macOS/Windows:** [Docker Desktop](https://www.docker.com/products/docker-desktop)
- **Linux:** [Docker Engine](https://docs.docker.com/engine/install/)

Docker Compose comes bundled with Docker Desktop. If you installed Docker Engine on Linux, also install [Docker Compose](https://docs.docker.com/compose/install/).

Check Docker and Docker Compose:

```bash
docker --version
docker compose version
```

**Check:** both commands print version numbers.

## Step 4 — Create an Anthropic API key

**What you're doing:** getting the key that lets your code call Claude and pay for it. Every API request carries this key, so treat it like a password.

1. Sign in at console.anthropic.com (separate from the Claude chat app).
2. Add prepaid credit; $10–20 covers Modules 0–4 on Haiku.
3. Create an API key. It starts with `sk-ant-` and is shown only once. Paste it somewhere temporary (a password manager) — you'll paste it into `.env` in Step 6.

**Check:** the key shows as active in the Console.

## Step 5 — Set a spend limit

**What you're doing:** capping how much the API can charge you in a month. It's a safety net: a bug can't cost more than the limit you set.

In the Console's billing/limits settings, set a monthly limit slightly above your credit. If a bug in a loop fires thousands of requests, the API refuses them instead of draining your balance.

**Check:** the limit is visible in the Console.

## Step 6 — Create `.env` with your API key and database passwords

**What you're doing:** creating `.env`, the one private file that holds your secrets: your Anthropic API key and three database passwords. A command writes the file, so the format is always right and no password is typed or pasted by hand.

Run this in the project folder. It will ask you to paste your API key from Step 4:

```bash
(
  if [ -e .env ]; then echo ".env already exists, left unchanged"; exit; fi
  printf 'Paste your API key (starts with sk-ant-), then press Enter: '
  read -rs key; echo
  p1=$(openssl rand -hex 24); p2=$(openssl rand -hex 24); p3=$(openssl rand -hex 24)
  printf '%s\n' \
    "ANTHROPIC_API_KEY=$key" \
    "MONGO_ROOT_PASSWORD=$p1" \
    "MONGODB_URI_RW=mongodb://course_rw:$p2@127.0.0.1:27017/course?authSource=admin&directConnection=true" \
    "MONGODB_URI=mongodb://course_ro:$p3@127.0.0.1:27017/course?authSource=admin&directConnection=true" > .env
  chmod 600 .env
  echo "wrote .env"
)
```

**Check:** prints `wrote .env`. If it prints `.env already exists, left unchanged`, you created `.env` before: keep it, because your database users use the passwords inside it.

What the command does:

- Reads your API key from stdin and includes it directly in `.env`.
- `openssl rand -hex 24` makes each database password: 48 characters of `0-9` and `a-f`, strong and with no symbols that could break a connection string.
- Writes five settings, one per line, ending with a line break.
- `chmod 600` lets only your user account read the file.
- The brackets keep the passwords out of your shell once the command ends; they exist only in `.env`.

| Setting | Value | Used for |
| --- | --- | --- |
| `ANTHROPIC_API_KEY` | Your API key from Step 4 | Claude API calls |
| `MONGO_ROOT_PASSWORD` | Random password 1 | MongoDB admin account (only for creating users) |
| `MONGODB_URI_RW` | Random password 2 | `course_rw`, the read-write user for your scripts |
| `MONGODB_URI` | Random password 3 | `course_ro`, the read-only user for Claude's queries |

- The plain `MONGODB_URI` is the **read-only** user, so the default is the safe one.
- `authSource=admin` tells MongoDB where the users are stored.
- `directConnection=true` is needed because MongoDB runs as a one-machine replica set.

Verify the four names are in place:

```bash
cut -d= -f1 .env
```

**Check:** prints `ANTHROPIC_API_KEY`, `MONGO_ROOT_PASSWORD`, `MONGODB_URI_RW` and `MONGODB_URI`, one per line.

Make a template with the same names but no values, which you can commit:

```bash
sed 's/=.*/=/' .env > .env.example
```

Confirm git will never commit `.env`:

```bash
git check-ignore .env
```

**Check:** prints `.env`.

## Step 7 — Test the API key with curl

**What you're doing:** sending one tiny message to Claude with the simplest possible tool, to prove your key and credit work before any Docker containers are running.

This sends one tiny request straight to the API using the key in `.env`, with no Docker involved. If it works, any later problem is in the container, not your key.

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

## Step 8 — Build and start the Docker containers

**What you're doing:** building the Python application container and starting both the Python app and MongoDB database. The containers read their configuration from `.env`.

The `docker-compose.yml` file is included in the repo. It defines:
- **mongodb**: The database with search engine (`mongot`)
- **python_app**: Your Python environment with all course packages pre-installed

Build and start the containers:

```bash
docker compose up -d
```

**Check:** the first time, Docker downloads the MongoDB image and builds the Python image (this can take a few minutes). It prints lines ending with `mongodb-1  Started` and `python_app-1  Started`.

What this does:

- Builds the Python container with all dependencies from `requirements.txt` (anthropic, pymongo, pandas, etc.)
- Runs MongoDB **and** its search engine (`mongot`, needed for full-text search in Module 9) in one container.
- `MONGODB_INITDB_ROOT_PASSWORD` reads from `.env` (Step 6); only that one value goes into the container.
- Volumes preserve your data and code when containers restart.
- The Python app container mounts your project directory, so changes to code appear immediately inside the container.

**Check:** wait about a minute, then run:

```bash
docker compose ps
```

Output shows `mongodb` as **healthy**. If it says `starting`, wait and check again.

Check that mongosh runs inside the container:

```bash
docker compose exec mongodb mongosh --version
```

**Check:** prints a version number such as `2.12.0`.

## Step 9 — Check the admin login

**What you're doing:** logging in to MongoDB as the admin with password 1, to prove the database is running and the password from `.env` reached it. You'll need this login in the next step to create the users.

Print password 1 (the admin password) from `.env`, so you can copy it:

```bash
grep '^MONGO_ROOT_PASSWORD=' .env | cut -d= -f2-
```

**Check:** prints one line of 48 letters and digits.

Log in as the admin:

```bash
docker compose exec mongodb mongosh "mongodb://admin@127.0.0.1:27017/admin?directConnection=true"
```

`docker compose exec mongodb` runs mongosh inside the MongoDB container. There, `127.0.0.1` is the container itself, where MongoDB listens, so the same addresses as in `.env` work.

When asked for a password, paste password 1. You should see a prompt like:

```
AtlasLocalDev mongodb [direct: primary] admin>
```

Type `exit` to leave mongosh and get your normal terminal back; the next step runs there.

**Check:** you reached the prompt above without "Authentication failed".

## Step 10 — Create the two users

**What you're doing:** creating the two database users your code will log in as: `course_rw`, which can read and write, and `course_ro`, which can only read. Keeping a read-only user is what later makes it safe to run queries that Claude writes.

This command reads passwords 2 and 3 from `.env` and creates both users. It first removes either user if it already exists, so it is safe to run again.

```bash
(
  get() { grep "^$1=" .env | cut -d= -f2- | tr -d '\r'; }
  pw()  { get "$1" | sed -E 's#^mongodb://[^:]+:([^@]+)@.*#\1#'; }
  export RW_PW="$(pw MONGODB_URI_RW)" RO_PW="$(pw MONGODB_URI)"
  docker compose exec -e RW_PW -e RO_PW mongodb mongosh "mongodb://admin@127.0.0.1:27017/admin?directConnection=true" --quiet --eval '
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

When asked, paste **password 1** (admin). Step 9 shows how to print it from `.env`.

How it works:

1. `get` reads one line from `.env` and strips Windows line endings.
2. `pw` cuts the password out of each connection string (between `user:` and `@`).
3. The passwords reach mongosh as environment variables that exist only inside the brackets; `-e RW_PW -e RO_PW` passes them into the container. They never appear on screen or in a history file.

> **Don't paste passwords into `passwordPrompt()` inside mongosh.** Terminals add hidden characters to pasted text, and MongoDB rejects the password with `U_STRINGPREP_PROHIBITED_ERROR`, however simple or strong it is.

What you now have:

- **course_rw** — can read and write the `course` database. Your scripts use it.
- **course_ro** — can only read the `course` database. No inserts, updates, deletes, `$out` or `$merge`. Claude's queries use it.

**Check:** the output ends with `users created`. On a rerun, `dropped course_rw` and `dropped course_ro` come first; that's expected.

## Step 11 — Check both users

**What you're doing:** logging in with each connection string from `.env` and asking MongoDB what that user is allowed to do. This proves both passwords work and that the read-only user really is read-only.

Log in as the read-only user (`MONGODB_URI`):

```bash
docker compose exec mongodb mongosh "$(grep '^MONGODB_URI=' .env | cut -d= -f2-)" --quiet --eval "db.runCommand({connectionStatus: 1}).authInfo.authenticatedUserRoles"
```

**Check:** prints `[ { role: 'read', db: 'course' } ]`.

Log in as the read-write user (`MONGODB_URI_RW`):

```bash
docker compose exec mongodb mongosh "$(grep '^MONGODB_URI_RW=' .env | cut -d= -f2-)" --quiet --eval "db.runCommand({connectionStatus: 1}).authInfo.authenticatedUserRoles"
```

**Check:** prints `[ { role: 'readWrite', db: 'course' } ]`.

If either says "Authentication failed": open `.env`, look for spaces or quotes around values, fix them, and repeat Step 11.

## Step 12 — Prove everything from Python in a notebook

**What you're doing:** checking from Python, the way every later module works, that the pieces fit together: the key in `.env` reaches Claude, the read-write user can save a document, the read-only user can read it, and the read-only user is blocked from writing. The notebook you save is your proof that setup is complete.

Make a folder for notebooks:

```bash
mkdir -p notebooks
```

Start Jupyter inside the Docker container:

```bash
docker compose exec python_app jupyter notebook --ip=0.0.0.0 --port=8888 --no-browser
```

**Check:** the output prints a URL starting with `http://127.0.0.1:8888/?token=`. Copy and paste it into your browser. Jupyter keeps running in this terminal until you stop it: when you've saved the notebook, press Ctrl+C twice in the terminal (or open a second terminal for Step 13).

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

## Step 13 — Commit your setup files

**What you're doing:** saving your setup files to your fork on GitHub, after checking that no secret is in them. From here on, every module ends with a commit like this one.

Commit the files that are safe to share. Never `.env`. Run these one at a time.

See what git would pick up:

```bash
git status --short
```

**Check:** lists `?? .env.example` and `?? notebooks/`. `.env` and `docker-compose.yml` must NOT be listed (docker-compose.yml is already in the repo).

Search the three files you are about to commit for an API key or a connection string with a password. Notebook outputs are saved inside the file, so this also catches a cell that printed a secret:

```bash
grep -n "sk-ant-\|mongodb://[^ ]*:[^ ]*@" docker-compose.yml .env.example notebooks/m00_setup.ipynb || echo "no secrets found"
```

**Check:** prints `no secrets found`. If it prints a line instead, remove that secret (for a notebook, clear the cell's output and save) and run it again.

Stage the files:

```bash
git add .env.example notebooks/m00_setup.ipynb
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
