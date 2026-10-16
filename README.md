# GitHub Actions — continuous deployment to the VPS

This folder holds the **reusable deployment workflow** shared by every
project, plus an example of the small caller file each project repo
contains. On every `git push` to `main`, the project's code is built (on the
GitHub runner or on the VPS, depending on the stack), copied to the VPS with
`rsync`, and its systemd service is restarted.

> Where it fits in the overall infrastructure:
>
> | Piece | Responsibility | How often |
> |---|---|---|
> | `infra/terraform/` | OVH DNS: `bylucas.eu` and `*.bylucas.eu` → VPS IP | Once |
> | `infra/ansible/` | Everything on the VPS: folder, `.env`, systemd service, nginx vhost, TLS certificate | Once per project (and when the VPS is rebuilt) |
> | **`infra/github-actions/`** (here) | Shipping the **code** and restarting the service | On every push to `main` |
>
> **The CD creates nothing.** The project must have been declared in Ansible
> (`group_vars/all/vars.yml`) and the playbook run **before** the first
> deployment, otherwise the target folder, the `.env`, the systemd unit and
> the restart permissions don't exist.

---

## Table of contents

1. [Layout and the "central repo" model](#1-layout-and-the-central-repo-model)
2. [The caller workflow (in each project)](#2-the-caller-workflow-in-each-project)
3. [The reusable workflow: inputs and secrets](#3-the-reusable-workflow-inputs-and-secrets)
4. [Step-by-step execution](#4-step-by-step-execution)
5. [What happens for each project type](#5-what-happens-for-each-project-type)
6. [How SSH and sudo work](#6-how-ssh-and-sudo-work)
7. [Contract with Ansible](#7-contract-with-ansible)
8. [Setup from scratch](#8-setup-from-scratch)
9. [Adding the CD to a new project](#9-adding-the-cd-to-a-new-project)
10. [Debugging a failed deployment](#10-debugging-a-failed-deployment)
11. [Limitations and known pitfalls](#11-limitations-and-known-pitfalls)

---

## 1. Layout and the "central repo" model

```
github-actions/
├── .github/workflows/
│   └── reusable-deploy.yml     # The reusable workflow (all the deployment logic)
├── exemple-projet-deploy.yml   # Template of the caller file to copy into each project
└── README.md
```

This folder is the source of a **separate GitHub repository**,
`LF2Zetrei/infra-github-actions`. GitHub only resolves reusable workflows
from a repository, at the path `.github/workflows/`. Projects reference it
like this:

```yaml
uses: LF2Zetrei/infra-github-actions/.github/workflows/reusable-deploy.yml@main
```

```
   project repo (landing, monitoring, catlendar...)       LF2Zetrei/infra-github-actions
 ┌───────────────────────────────────────────┐           ┌──────────────────────────────┐
 │ .github/workflows/deploy.yml  (~15 lines) │──uses────▶│ .github/workflows/           │
 │   on: push to main                        │           │   reusable-deploy.yml        │
 │   with: project_name/type/remote_path     │           │   (build + rsync + restart)  │
 │   secrets: VPS_HOST/USER/SSH_KEY          │           └──────────────────────────────┘
 └───────────────────────────────────────────┘
```

Why: the deployment logic (~150 lines) lives in **one place**. Fixing or
improving it fixes every project at the next push, without touching the
project repos.

> ⚠️ Editing `reusable-deploy.yml` in **this** folder has no effect until it
> is pushed to the `main` branch of `LF2Zetrei/infra-github-actions`. Every
> project uses `@main`, so a push there affects **all** projects immediately.

---

## 2. The caller workflow (in each project)

File `.github/workflows/deploy.yml` in each project repo (based on
`exemple-projet-deploy.yml`). Real example from `monitoring`:

```yaml
name: Deploy

on:
  push:
    branches: [main]          # trigger: every push (or merged PR) to main

jobs:
  deploy:
    uses: LF2Zetrei/infra-github-actions/.github/workflows/reusable-deploy.yml@main
    with:
      project_name: monitoring                    # = "name" in Ansible's projects
      project_type: node                          # = "type" in Ansible's projects
      remote_path: /home/lucas/apps/monitoring    # = "path" in Ansible's projects
    secrets:
      VPS_HOST: ${{ secrets.VPS_HOST }}
      VPS_USER: ${{ secrets.VPS_USER }}
      VPS_SSH_KEY: ${{ secrets.VPS_SSH_KEY }}
```

Secrets are not inherited automatically by a reusable workflow: they must
be passed explicitly as above (or with `secrets: inherit`).

Current callers in this workspace:

| Repo | project_name | project_type | remote_path |
|---|---|---|---|
| `landing` | `landing` | `static` | `/var/www/landing` |
| `monitoring` | `monitoring` | `node` | `/home/lucas/apps/monitoring` |

---

## 3. The reusable workflow: inputs and secrets

Trigger: `on: workflow_call` (it can only be called by another workflow).

### Inputs

| Input | Type | Accepted values | Used for |
|---|---|---|---|
| `project_name` | string | Ansible `name` | `systemctl restart <project_name>` / `is-active` |
| `project_type` | string | `node`, `python`, `rust`, `springboot`, `static` | Chooses which build/sync/install steps run (`if:` conditions) |
| `remote_path` | string | Ansible `path` | rsync destination and working dir of the remote commands |

### Secrets

| Secret | Content | Used for |
|---|---|---|
| `VPS_HOST` | VPS IP or domain (e.g. `bylucas.eu`) | `ssh-keyscan`, `rsync`, `ssh` targets |
| `VPS_USER` | `lucas` | SSH/rsync user |
| `VPS_SSH_KEY` | Content of the **private** deploy key (`-----BEGIN OPENSSH PRIVATE KEY-----...`) | Loaded into the runner's `ssh-agent` |

### Execution environment

One job, `build-and-deploy`, on `ubuntu-latest` (a fresh GitHub-hosted VM
per run, destroyed afterwards). Steps run sequentially; the first failing
step stops the job (the following steps are skipped) and the run turns red.

---

## 4. Step-by-step execution

Every step has an `if:` on `project_type`; the ones that don't apply show as
"skipped" in the GitHub UI.

| # | Step | Runs for | Where | What it does |
|---|---|---|---|---|
| 1 | Checkout | all | runner | `actions/checkout@v4`: clones the pushed commit into the runner |
| 2 | Setup Node | `static` | runner | `actions/setup-node@v4`, Node 20 |
| 3 | Build Next.js (static export) | `static` | runner | `npm ci && npm run build` → produces `out/` |
| 4 | Setup Rust | `rust` | runner | `actions-rs/toolchain@v1`, stable toolchain |
| 5 | Build Rust | `rust` | runner | `cargo build --release` → `target/release/<binary>` |
| 6 | Setup Java | `springboot` | runner | `actions/setup-java@v4`, Temurin 21 |
| 7 | Build Spring Boot | `springboot` | runner | `./mvnw clean package -DskipTests=false` (tests **do** run, a failing test blocks the deploy) → `target/*.jar` |
| 8 | Add SSH key | all | runner | `webfactory/ssh-agent@v0.9.0` loads `VPS_SSH_KEY` into an ssh-agent |
| 9 | Add VPS to known hosts | all | runner | `ssh-keyscan -H $VPS_HOST >> ~/.ssh/known_hosts` |
| 10 | Sync static site | `static` | runner → VPS | `rsync -avz --delete out/ user@host:<remote_path>/` |
| 11 | Sync project with a process | not `static` | runner → VPS | `rsync -avz --delete --exclude .git --exclude node_modules --exclude .env ./ user@host:<remote_path>/` |
| 12 | Create Python venv if missing | `python` | VPS | `test -d <path>/venv \|\| python3 -m venv <path>/venv` |
| 13 | Ensure pip in venv | `python` | VPS | `<path>/venv/bin/python3 -m ensurepip --upgrade` |
| 14 | Install Python deps | `python` | VPS | `<path>/venv/bin/pip install -r <path>/requirements.txt` |
| 15 | Install Node deps and build | `node` | VPS | `cd <path> && npm ci && npm run build` |
| 16 | Restart service | not `static` | VPS | `sudo systemctl restart <project_name>` |
| 17 | Check service is running | not `static` | VPS | `systemctl is-active --quiet <project_name>` — non-zero exit → job fails |

### About rsync `--delete` and `--exclude`

- `--delete` removes from the VPS every file that no longer exists in the
  repo → the VPS is an exact mirror of the commit (no stale files).
- `--exclude` patterns are **protected** from deletion on the VPS side
  (rsync doesn't touch excluded files unless `--delete-excluded` is used):
  - `.env` → the file generated by Ansible survives every deployment;
  - `node_modules` → kept between deployments (`npm ci` reinstalls it anyway);
  - `.git` → not sent.
- Everything else present only on the VPS is deleted at each deployment —
  including build artefacts (`.next/`) and the Python `venv/` (see §11).

---

## 5. What happens for each project type

### `static` (e.g. `landing`, `portfolio`)

```
runner: checkout → Node 20 → npm ci → npm run build (Next.js "output: 'export'") → out/
rsync out/  →  /var/www/<project>/      (exact mirror, --delete)
no service, no restart: nginx serves the files directly, the change is live immediately
```

Requirements: `npm run build` must produce an `out/` folder (Next.js with
`output: 'export'` in `next.config`). The folder `/var/www/<project>` must
exist and belong to `lucas` (created by Ansible).

### `node` (e.g. `monitoring`, `catlendar`, `warp10v2`)

```
runner: checkout (no build on the runner)
rsync whole repo (minus .git, node_modules, .env)  →  /home/lucas/apps/<project>/
VPS:    cd <path> && npm ci && npm run build
VPS:    sudo systemctl restart <project>   → ExecStart=/usr/bin/npm run start
VPS:    systemctl is-active <project>
```

Why build **on the VPS** rather than on the runner: building on one machine
and running on another caused incompatibilities in production (Node
version, architecture, Turbopack module hashes). Building where it runs
avoids that.

`npm ci` installs devDependencies too (needed by the build: TypeScript,
Tailwind/PostCSS...). They stay on the VPS; harmless.

The build runs in `<path>`, where Ansible's `.env` lives: Next.js reads it
at build time, which matters for `NEXT_PUBLIC_*` variables (inlined into
the bundle at build time — see `NEXT_PUBLIC_BASE_URL` in `warp10v2`).

### `python`

```
runner: checkout
rsync whole repo  →  /home/lucas/apps/<project>/
VPS:    create venv/ if missing → ensurepip → pip install -r requirements.txt
VPS:    sudo systemctl restart <project>   → ExecStart=<path>/venv/bin/python3 <entrypoint>
VPS:    systemctl is-active <project>
```

Requirements: a `requirements.txt` at the repo root; `python3-venv` on the
VPS (installed by Ansible).

### `rust`

```
runner: checkout → stable toolchain → cargo build --release
rsync whole repo, including target/  →  <path>/
VPS:    sudo systemctl restart <project>   → ExecStart=<path>/target/release/<binary>
```

The binary is compiled on the runner (`ubuntu-latest`) and executed on the
VPS: both must have a compatible glibc (the VPS's Ubuntu version must not be
older than the runner's).

### `springboot`

```
runner: checkout → Temurin 21 → ./mvnw clean package (with tests)
rsync whole repo, including target/  →  <path>/
VPS:    sudo systemctl restart <project>   → ExecStart=/usr/bin/java -jar <path>/target/<jar|app.jar>
```

Requirements: `mvnw` committed and executable; the jar name must match
Ansible's `jar` field (default `app.jar` → set `<finalName>app</finalName>`
in the `pom.xml`, or set `jar:` in Ansible); a Java 21 runtime must be
installed on the VPS (**not** done by Ansible today).

### `c`

Ansible knows the `c` type, but the reusable workflow has **no build step**
for it and its `description` doesn't list it. With `project_type: c`, the
repo is rsynced and the service restarted, so the compiled binary would
have to be committed to the repo. Add a `make` step before using it.

---

## 6. How SSH and sudo work

### Authentication

1. A dedicated SSH key pair (`deploy_key`), separate from your personal key.
2. Public key in `/home/lucas/.ssh/authorized_keys` on the VPS.
3. Private key in the GitHub secret `VPS_SSH_KEY`.
4. At runtime, `webfactory/ssh-agent` loads it into an agent; `rsync` and
   `ssh` use it transparently.

The VPS only accepts key authentication (Ansible's `ssh-hardening` disables
passwords and root login); fail2ban bans IPs after 5 failures — a
misconfigured key can get the GitHub runner's IP banned for 1 h (each run
gets a different IP, so it's rarely noticeable).

### Host verification

`ssh-keyscan` fetches the VPS host key at each run and trusts it
("trust on first use"). Simple, but it doesn't protect against a spoofed
server. To harden: store the output of `ssh-keyscan <host>` in a
`VPS_KNOWN_HOSTS` secret and write that into `~/.ssh/known_hosts` instead.

### Passwordless sudo

The workflow runs `sudo systemctl restart <name>`, non-interactively: sudo
must not ask for a password. The file `/etc/sudoers.d/monitoring-restart`
**generated by Ansible** (`app-deploy` role) already contains, for every
non-static project:

```
lucas ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart <name>
```

which matches the command run by the workflow exactly. `systemctl is-active`
doesn't need sudo.

So no manual `visudo` is required as long as the project was declared in
Ansible and the playbook was run. (An older manual alternative was a
`/etc/sudoers.d/lucas-systemctl` file with
`NOPASSWD: /usr/bin/systemctl restart *` — broader, so less safe; prefer
the Ansible-generated one.)

---

## 7. Contract with Ansible

For a deployment to work, these values must line up exactly:

| GitHub Actions (caller) | Ansible (`group_vars/all/vars.yml`) | Why |
|---|---|---|
| `project_name` | `name` | systemd unit name + sudoers entry |
| `project_type` | `type` | `static` → no service; otherwise the matching systemd template |
| `remote_path` | `path` | rsync target = `WorkingDirectory` of the service / nginx `root` |

And these must already exist on the VPS (all created by Ansible):

| Item | Created by |
|---|---|
| `<remote_path>/` owned by `lucas` | `app-deploy` |
| `<remote_path>/.env` | `app-deploy` (from `env_vars`) |
| `/etc/systemd/system/<name>.service` (enabled) | `app-deploy` |
| sudoers entry `systemctl restart <name>` | `app-deploy` |
| nginx vhost + TLS certificate | `app-deploy` + Certbot |
| Node.js / python3-venv | `app-deploy` |

Division of labour:

| | Ansible | GitHub Actions |
|---|---|---|
| Code, `node_modules`, venv, build output | — | ✅ |
| `.env` (secrets) | ✅ | never touches it (excluded) |
| systemd unit, nginx, certificate, DB | ✅ | — |
| Restart after a code change | — | ✅ |
| Restart after an `.env` change | ❌ (do it by hand or push) | ✅ at the next push |

---

## 8. Setup from scratch

### 1. Create a dedicated deploy key

```bash
ssh-keygen -t ed25519 -f ~/.ssh/deploy_key -N "" -C "github-actions-deploy"
```

### 2. Authorise it on the VPS

```bash
ssh-copy-id -i ~/.ssh/deploy_key.pub -o IdentityFile=~/.ssh/bylucas lucas@bylucas.eu
```

(`-o IdentityFile` because password login is disabled: you need your
existing key to add a new one.)

### 3. Publish the central repo

Push the content of this folder to `LF2Zetrei/infra-github-actions`
(the workflow must end up at `.github/workflows/reusable-deploy.yml` in
that repo).

- **Public repo**: usable by every repo directly.
- **Private repo**: Settings → Actions → General → Access → "Accessible from
  repositories owned by the user 'LF2Zetrei'". Otherwise callers fail with
  `workflow was not found`.

### 4. Configure the secrets

In each project repo (Settings → Secrets and variables → Actions), or once
at the organisation level if the repos belong to one:

| Name | Value |
|---|---|
| `VPS_HOST` | `bylucas.eu` (or the IP) |
| `VPS_USER` | `lucas` |
| `VPS_SSH_KEY` | full content of `~/.ssh/deploy_key` (the **private** key, including the BEGIN/END lines) |

Quick upload with the GitHub CLI:

```bash
gh secret set VPS_HOST   -R LF2Zetrei/<repo> -b "bylucas.eu"
gh secret set VPS_USER   -R LF2Zetrei/<repo> -b "lucas"
gh secret set VPS_SSH_KEY -R LF2Zetrei/<repo> < ~/.ssh/deploy_key
```

### 5. Sudo rights

Nothing to do: generated by Ansible (see §6).

---

## 9. Adding the CD to a new project

1. Declare the project in Ansible and run the playbook (see
   `infra/ansible/README.md` §9).
2. In the project repo, create `.github/workflows/deploy.yml` from
   `exemple-projet-deploy.yml`:
   - replace `ton-compte-github` with `LF2Zetrei`;
   - set `project_name`, `project_type`, `remote_path` to the Ansible values.
3. Add the three secrets (§8.4) if they aren't set at a shared level.
4. Check the type-specific requirements (§5): `out/` for static,
   `requirements.txt` for Python, a `build` + `start` script for Node...
5. `git push` to `main`, follow the run under the repo's **Actions** tab.
6. Check: `https://<subdomain>` answers; on the VPS
   `systemctl status <name>` is `active (running)`.

To redeploy without a code change: Actions → Deploy → "Re-run all jobs"
(add `workflow_dispatch:` under `on:` in the caller to get a "Run workflow"
button).

---

## 10. Debugging a failed deployment

| Failing step | Likely cause | Fix |
|---|---|---|
| `uses: ...reusable-deploy.yml@main` — "workflow was not found" | Central repo private without access, wrong owner/path | §8.3 |
| Add SSH key — "key is invalid" | `VPS_SSH_KEY` truncated, or public key pasted | Re-paste the full private key |
| rsync / ssh — `Permission denied (publickey)` | Deploy key not in `authorized_keys`, wrong `VPS_USER` | §8.2 |
| rsync — `No such file or directory` / permission denied on target | Folder doesn't exist or isn't owned by `lucas` | Run Ansible for that project |
| Sync static — `out/: No such file` | Next.js not configured with `output: 'export'` | Fix `next.config` |
| `npm ci` on VPS fails | Missing/out-of-sync `package-lock.json`, VPS out of RAM | Commit the lockfile; check RAM/swap |
| Restart — `sudo: a password is required` | No sudoers entry for that `project_name` (not declared in Ansible, or name mismatch) | Align names and re-run Ansible |
| Restart — `Unit <name>.service not found` | Service never created by Ansible | Run Ansible for that project |
| Check `is-active` fails | The app crashed at startup (bad `.env`, missing dependency, port already in use...) | On the VPS: `journalctl -u <name> -n 100 --no-pager` |

Note that `is-active` is checked immediately after the restart: an app that
crashes a few seconds after starting can still show green. Use
`journalctl -u <name> -f` on the VPS to be sure.

---

## 11. Limitations and known pitfalls

- **No tests** before deployment (except Spring Boot via Maven). To add for
  Node: a `npm test` step before the deployment steps; any failing step
  stops the job, so nothing gets deployed.
- **No rollback**: if the new version doesn't start, the service stays down
  until the next push. Fix forward, or `git revert` + push.
- **Downtime during Node deployments**: rsync `--delete` removes `.next/`
  while the old process is still running, then `npm run build` takes a
  while before the restart. The site can return errors for the duration of
  the build. A fix: build into a separate directory and swap, or exclude
  `.next` from rsync.
- **Python venv recreated every time**: `venv/` is not in the rsync excludes,
  so `--delete` removes it on each deployment; the next step recreates it
  and reinstalls everything. It works but is slow. Add `--exclude 'venv'` to
  keep it.
- **No concurrency control**: two quick pushes can run two deployments at
  the same time against the same folder. Add to the reusable workflow:
  `concurrency: { group: deploy-${{ inputs.project_name }}, cancel-in-progress: false }`.
- **`@main` everywhere**: a breaking change to the reusable workflow hits
  every project at once. Use a tag (`@v1`) in callers for more control.
- **`actions-rs/toolchain@v1` is archived** (no longer maintained, uses a
  deprecated Node runtime). Replace with `dtolnay/rust-toolchain@stable`.
- **`project_type: c`** isn't handled (see §5).
- **`.env` changes**: Ansible rewrites the `.env` but doesn't restart; the
  new values are only picked up at the next deployment or a manual restart.
