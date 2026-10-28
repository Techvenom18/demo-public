# Progress Update — 27 October 2026

**Who:** Sumit (Junior Backend Dev) · **Task:** Port the Jenkins CI/CD pipeline
to GitHub Actions (Method B: SSH + rsync) for `demo-public` and `demo-private`.

---

## What we accomplished today

### 1. Documentation (3 explanation docs + 1 plan)

| File | Repo / branch | Purpose | Status |
|---|---|---|---|
| `docs/CICD-PIPELINE-EXPLAINED.md` | demo-public · committed earlier (`b076952`) | Full pipeline behavior with flowcharts: 4 stages, 5 protection layers, backup rotation, rsync excludes, rollback sequence, concurrency | ✅ on `ci/github-actions` |
| `docs/GITHUB-ACTIONS-STUDY-GUIDE.md` | demo-public · main (`1d5e0ed`) + now also on `ci/github-actions` | Beginner → Pro study guide: YAML, events, jobs/steps, secrets, environments, concurrency, security, debugging, 10 exercises | ✅ pushed |
| `docs/Workflow-Explanation.md` | demo-public · `ci/github-actions` | **The implementation plan**: Method A vs B decision, both-repo matrix, 5-layers→YAML mapping, 5-stage rollout, secrets/variables names, open questions | ✅ pushed (`488d822`) |
| `docs/Private-Workflow-Explanation.md` | demo-private · `ci/github-actions` | Line-by-line explanation of the private repo's `deploy.yml` with diagrams and Jenkins mappings | ✅ pushed (`bf256f4`) |

### 2. Workflows written

| File | Repo | Contents | Status |
|---|---|---|---|
| `.github/workflows/deploy.yml` (277 lines) | demo-public | **2 jobs = 3 stages.** `build-test` (ACTIVE: checkout → node → npm ci → build → test → detect guard). `deploy` (written, not yet usable: config guard → SSH setup → exclude list → version.txt → Layer 2 backup keep-5 → rsync upload → Layer 4 health checks → Layer 4 rollback `if: failure()` → Layer 5 alert → key cleanup) | ✅ pushed `488d822` |
| `.github/workflows/deploy.yml` (278 lines) | demo-private | Same pipeline, `demo-private` defaults, private-repo note (GITHUB_TOKEN, no deploy key needed) | ✅ pushed `bf256f4` |
| `.github/workflows/test-connection.yml` (133 lines) | demo-public | **Stage 3 tool:** 7-step connection check — secrets exist? → key/fingerprint load → TCP reach SSH port → cPanel port info → SSH login → rsync present? → summary. **Deploys nothing.** Manual button only | ⏳ written, NOT yet committed |
| `.github/workflows/ci.yml` | demo-public · main | Old CI only (still on `main`; to be **removed when we merge** deploy.yml — its steps live on inside `build-test`) | ⏸ untouched |

### 3. Git activity today

```
demo-public   branch ci/github-actions  → PUSHED (new branch)
              commit 488d822  deploy.yml + Workflow-Explanation.md + study guide

demo-private  branch ci/github-actions  → PUSHED (new branch)
              commit bf256f4  deploy.yml + Private-Workflow-Explanation.md

main          (both repos)  untouched — nothing merged, nothing deployed
```

### 4. Decisions made

1. **Method B chosen** (SSH + rsync from the runner) over Method A (cPanel
   API + Git Version Control) — keeps the proven backup/health-check/rollback
   logic and doesn't require committing `dist/` to the repo.
2. **SSH keys exist but are deliberately NOT configured yet** — secrets get
   added in stage 3, on explicit approval.
3. **One unified `deploy.yml` replaces `ci.yml`** at merge time (no duplicate
   builds).
4. **`environment: demo` now; `production` later** with required reviewers.
5. **Concurrency = `deploy-<APP_NAME>` with `cancel-in-progress: false`** —
   a running deploy is never cancelled halfway.

### 5. All 5 protection layers implemented in both `deploy.yml` files

| Layer | Implementation |
|---|---|
| 1 — Git history | checkout; base recovery path |
| 2 — Backup | `cp -a live → backup-<run>`, keep newest 5, outside public_html |
| 3 — Excludes + guard | `.rsync-exclude` (.env, node_modules, tests… + `SERVER_ONLY`); detect step refuses empty uploads before any `--delete` |
| 4 — Health check + rollback | `test -f` entry file + curl `SITE_URL` (retries); `if: failure()` rsync backup back, job stays red |
| 5 — Safety rules | main-only `if`, PRs never deploy, concurrency lock, 15/20-min timeouts, config guard, `ALERT` echo |

---

## Current state (as of end of day)

```
demo-public     ci/github-actions → pushed: deploy.yml + 3 docs
                main → old ci.yml still there (removes at merge)
demo-private    ci/github-actions → pushed: deploy.yml + private doc
test-connection.yml → written in demo-public, NOT committed yet

Secrets:        NONE added yet (by design — stage 3 pending)
Variables:      NONE added yet
Environments:   none created yet (demo → stage 4)
Server:         untouched — zero contact so far
Jenkins:        still the last known deployer (disable only after Actions proves itself)
```

## Next steps (in order)

1. **Commit `test-connection.yml`** → push → appears in Actions tab.
2. **Collect BigRock facts** (offline): SSH host, port, user, demo folder
   path, demo URL; ask about **IP restriction on SSH** (the big unknown) and
   whether `rsync` is installed.
3. **Stage 3:** add 5 secrets (`SSH_PRIVATE_KEY`, `SSH_HOST`, `SSH_PORT`,
   `SSH_USER`, `SSH_KNOWN_HOSTS`) + variables (`APP_NAME`, `REMOTE_DIR`,
   `BACKUP_DIR`, `SITE_URL`, `SERVER_ONLY`) in **demo-public → Settings**;
   authorize the public key in cPanel; generate `SSH_KNOWN_HOSTS` via
   `ssh-keyscan`.
4. **Run "Test connection"** (manual button) and read the 7-step verdict:
   - all green → Method B confirmed
   - TCP step red → IP-restriction confirmed → escalate to lead
   - rsync step red → switch to lftp/SFTP fallback
5. **Stage 4:** merge PR to `main` (removes `ci.yml` in same PR — will show
   diff first), first deploy **to demo folder only**, create `demo` and
   `production` environments.
6. **Stage 5:** copy pipeline to demo-private (set `APP_NAME=demo-private`),
   later extract a reusable workflow for company repos, then disable old
   Jenkins jobs so nothing deploys twice.

## Open questions still unanswered

- [ ] BigRock SSH host / port / username
- [ ] `REMOTE_DIR` + `BACKUP_DIR` absolute paths
- [ ] Is `rsync` installed on the server?
- [ ] Does BigRock restrict SSH by IP? (GitHub runner IPs change every run)
- [ ] Demo folder path + demo URL for verification
- [ ] Server-only files list (`uploads/`, config) → fills `SERVER_ONLY`
- [ ] Where failure alerts should go (Slack/email — echo placeholder now)
