# Contributing to LMS

This doc walks you through getting the repo running locally, standing up local
infra with Podman, and the branch/PR workflow we use to get code into `master`.

---

## 1. Prerequisites

Install these before you start:

- [Git](https://git-scm.com/downloads)
- [Go](https://go.dev/dl/) (matching the version in `go.mod` once it's added to the repo)
- [Podman](https://podman.io/docs/installation) — used instead of Docker for local containers
- [VS Code](https://code.visualstudio.com/) (recommended IDE, used across the team)
- [GitHub CLI](https://cli.github.com/) (`gh`) — optional but makes raising PRs from the terminal easier

Verify Podman is working after install:

```bash
podman --version
podman info
```

If you hit PATH issues on macOS/Windows after installing Podman, that's common —
fixing it is usually a quick search away.

---

## 2. Get the repo locally

Clone the repo directly:

```bash
git clone https://github.com/salvatoreOm/lms.git
cd lms
```

Set your commit identity if you haven't already (use your real name/email so
commits are attributable):

```bash
git config user.name "Your Name"
git config user.email "you@example.com"
```

---

## 3. Local infrastructure with Podman

The stack runs on: **Postgres** (primary DB), **Redis** (caching), **RabbitMQ**
(message broker), and **Azurite** (local Azure Blob Storage emulator). All
containers share one Podman network so they can talk to each other and to the
Go app.

### 3.1 Create the shared network (once)

```bash
podman network create lms-network
```

### 3.2 Postgres

```bash
podman pull docker.io/library/postgres:17

podman run -d \
  --name lms-postgres \
  --network lms-network \
  -e POSTGRES_USER=lms_user \
  -e POSTGRES_PASSWORD=lms_dev_password \
  -e POSTGRES_DB=lms_db \
  -p 5432:5432 \
  -v lms-postgres-data:/var/lib/postgresql/data \
  docker.io/library/postgres:17
```

- DB name: `lms_db`
- Port: `5432`

Verify:

```bash
podman ps
```

### 3.3 Redis

```bash
podman pull docker.io/library/redis:8

podman run -d \
  --name lms-redis \
  --network lms-network \
  -p 6379:6379 \
  -v lms-redis-data:/data \
  docker.io/library/redis:8 \
  redis-server --appendonly yes
```

Verify:

```bash
podman exec lms-redis redis-cli ping   # should return PONG
```

- Redis port: `6379`
- Container name: `lms-redis`

### 3.4 RabbitMQ (message broker)

```bash
podman pull docker.io/library/rabbitmq:4-management

podman run -d \
  --name lms-rabbitmq \
  --network lms-network \
  -p 5672:5672 \
  -p 15672:15672 \
  -v lms-rabbitmq-data:/var/lib/rabbitmq \
  -e RABBITMQ_DEFAULT_USER=lms_user \
  -e RABBITMQ_DEFAULT_PASS=lms_dev_password \
  docker.io/library/rabbitmq:4-management
```

Verify at the management UI: [http://localhost:15672](http://localhost:15672)
(log in with the user/pass above).

### 3.5 Azurite (local Azure Blob Storage)

```bash
podman pull mcr.microsoft.com/azure-storage/azurite

podman run -d \
  --name lms-azurite \
  --network lms-network \
  -p 10000:10000 \
  -p 10001:10001 \
  -p 10002:10002 \
  -v lms-azurite-data:/data \
  mcr.microsoft.com/azure-storage/azurite
```

Verify:

```bash
podman ps
podman logs lms-azurite
```

- Blob Storage port: `10000`
- Queue Storage port: `10001`
- Table Storage port: `10002`

### 3.6 Everyday commands

```bash
podman ps                 # see what's running
podman stop <name>        # stop a container
podman start <name>       # start it again (data persists in the named volume)
podman logs -f <name>     # tail logs
```

You only need to `podman run ...` once per container — after that, `podman
start`/`stop` reuses the same container and its volume.

### 3.7 Local environment variables

Never commit real secrets. Copy `.env.example` (once it exists in the repo) to
`.env` and fill in local values matching the containers above — `.env` must
stay in `.gitignore`.

```bash
cp .env.example .env
```

---

## 4. Branch naming convention

Branch off `master` for every change. Branch names follow this pattern:

```text
lms-<number>
```

e.g. `lms-1`, `lms-2`, `lms-3`, ... — one branch per task/ticket, numbered
sequentially. Don't reuse a number once it's been used, even if that PR was
closed without merging.

```bash
git checkout master
git pull origin master
git checkout -b lms-12
```

Always branch from an up-to-date `master` to minimize conflicts later.

---

## 5. Making changes and committing

Keep commits focused and only stage what you actually changed — never blanket
`git add .` or `git add -A`, since that can accidentally pick up build
artifacts, local env files, or unrelated edits sitting in your working tree.

```bash
git status                  # see what changed
git diff                    # review the actual changes
git add path/to/file1.go path/to/file2.go   # stage only what you touched
git status                  # confirm the staged set is exactly what you expect
```

Before committing, double check you haven't staged anything that looks like a
secret (`.env`, credentials, API keys) even if the filename looks innocuous.

Write commit messages that explain *why*, not just *what*:

```bash
git commit -m "lms-12: add attendance tracking endpoint"
```

If `master` has moved on since you branched, bring your branch up to date
before raising a PR:

```bash
git checkout master
git pull origin master
git checkout lms-12
git rebase master        # or: git merge master, if your team prefers merge commits
```

Resolve any conflicts locally — never force-push over teammates' work on a
shared branch. If you must force-push your own branch after a rebase, use
`--force-with-lease`, not `--force`:

```bash
git push --force-with-lease origin lms-12
```

---

## 6. Pushing and raising a PR

Push your branch:

```bash
git push -u origin lms-12
```

Pushing the branch is what kicks off the PR flow already configured on the
GitHub repo. Open the PR against `master`:

```bash
gh pr create --base master --head lms-12 --title "lms-12: add attendance tracking endpoint" --fill
```

(Or open it from the GitHub UI — GitHub shows a "Compare & pull request"
banner right after you push a new branch.)

A few rules:

- **Never push directly to `master`.** All changes land there through a
  reviewed PR.
- Keep PRs small and scoped to one branch/task — easier to review, easier to
  revert if something's wrong.
- Fill in the PR description: what changed and why, and how you tested it
  (e.g. ran locally against the Podman stack above).
- Address review comments with new commits on the same branch — don't force
  push over review history unless asked to squash/clean up before merge.
- Once merged, delete the branch (GitHub can do this automatically on merge):

```bash
git checkout master
git pull origin master
git branch -d lms-12
```

---

## 7. Quick checklist before opening a PR

- [ ] Branched off latest `master`, named `lms-<number>`
- [ ] Only relevant files staged (`git status` reviewed before commit)
- [ ] No secrets/`.env`/credentials in the diff
- [ ] Rebased/merged latest `master` in, conflicts resolved
- [ ] Ran the app locally against the Podman stack (Postgres/Redis/RabbitMQ/Azurite) and verified the change
- [ ] Commit messages are clear and reference the branch/ticket number
