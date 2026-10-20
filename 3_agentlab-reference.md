# agentlab: technical reference (file by file)

*Reference guide to every important folder and file in `ZorroEng/agentlab`, as of 15 Oct 2026. Built by reading the repo's own docs and code. Real customer and patient data is never quoted. Where the repo's docs disagree with each other, that is noted.*

*For engineers. For what each project is, how it's used, credentials and procedures, see `agentlab-guide.md`. For a one-page overview, see `agentlab-summary.html`.*

## Contents
1. Repository root: `AGENTS.md`, `.claude/`, `.github/`, `.env.example`, `.gitignore`, conventions
2. `departments/ai_dept/`: Initiative Shop, org decks, agent setup, people setup, investor materials
3. `departments/sales/`: Census Raccoon, Proposal Studio
4. `departments/operations/`: Enrollment Copilot, CRM Cockpit, customer-team projects
5. `departments/compliance/`: AI policies
6. `departments/general_admin/`: Systems Manager
7. `departments/production/` and `knowledge_management/`: Engineering Support, OniCall, Local Setup
8. `departments/cross/`: Zorro Deck
9. `deployable/`: planned hosting platform
10. Appendix: docs that are out of date

## The whole repo on one page
```
agentlab/
├── AGENTS.md                     scoping gate (ask department → module first)
├── .claude/settings.json         Claude permissions (very open)
├── .github/workflows/            CI: builds the Census Raccoon Mac app
├── .env.example, .gitignore
├── departments/
│   ├── ai_dept/
│   │   ├── ai_initiative_shop/   how AI ideas become supported tools
│   │   ├── ai_org_deck/          exec decks
│   │   ├── ai_projects_setup/    agent identities + standard project shape
│   │   ├── ai_workstation_setup/ role profiles, tools, Mac setup kit, lst scheduler
│   │   └── investor_materials/   AI story for investors
│   ├── sales/
│   │   ├── census_ingestion/     Census Raccoon (census → standard xlsx)
│   │   └── proposals/            Proposal Studio (HTML/PDF proposals)
│   ├── operations/
│   │   ├── carrier_enrollment/   Enrollment Copilot (Chrome extension)
│   │   ├── unified_crm_cockpit/  "Zong" Intercom dashboard
│   │   └── customer_teams/       4 ops-run agent projects (snapshots)
│   ├── compliance/               AI & data policies → Google Docs
│   ├── general_admin/
│   │   └── systems_manager/      vendor/systems web app
│   ├── production/
│   │   ├── engineering_support/  support triage, SLOs, OniCall extension
│   │   └── local_setup/          run zorro-ts / zorro-py locally
│   ├── cross/
│   │   └── zorro_deck/           deck spec → HTML/PPTX/Google Slides
│   └── knowledge_management/     "moved" stub
└── deployable/                   design for a shared hosting platform
```

---
# Part 1: Repository root

```
agentlab/
├── AGENTS.md            ← the scoping gate: first file every agent reads
├── .claude/             ← Claude Code settings for this repo
├── .github/             ← GitHub Actions (CI) workflows
├── .env.example         ← template for root-level API keys
├── .gitignore           ← what git must never track
├── departments/         ← all the business work, one folder per department
└── deployable/          ← design for a future shared hosting platform
```

## `AGENTS.md` (root)
The **scoping gate**. Every Claude Code session starts here. It is not an index of the repo; it is a set of instructions:

- **The scoping gate:** before any work, the agent must ask:
  1. "Which department are you working in today?" (`general_admin`, `compliance`, `ai_dept`, `marketing`, `sales`, `operations`, `production`, `cross`, or the root project `deployable`)
  2. "Which module or initiative within it?"

  Then it loads that module's `AGENTS.md` and works only inside it.
- **A department table** with the entry file for each department.
- **Guardrails:**
  - Stay inside the declared module.
  - Everything else is read-only by default.
  - Shared files (root `AGENTS.md`, `.claude/`, `.github/`, root `.env*` and `.gitignore`, other departments' folders) need explicit approval for each edit.
- **"Moved" notes:** `knowledge_management`, the `zorro_html_deck_template` skill, and most of `production` now live in **ZorroEng/agent-production-hub**.
- `marketing` is listed but **reserved**: it has no folder yet.

## `.claude/settings.json`
Claude Code's permission settings for anyone opening the repo:
- **Allows without asking:** `Bash(*)`, `Read(*)`, `Write(*)`, `Edit(*)`, `Agent(*)`, `Workflow(*)`.
- `subagents.autoApprove: true` and `inheritPermissions: true`: helper agents get the same freedom.
- `.claude/settings.local.json` (personal overrides) is gitignored.

⚠️ This is very permissive. Claude can run any command and edit any file without a prompt. The safety comes from the written rules in `AGENTS.md`, not from permissions.

## `.github/workflows/build-census-raccoon-bundle.yml`
The **only CI workflow** in the repo. It builds the **Census Raccoon** Mac app for sales.

- **Triggers:** push to `main` that changes `sales/census_ingestion/census_ui/**`, `sales/census_ingestion/harness/**` or the workflow itself. It can also be run manually (`workflow_dispatch`).
- **Concurrency:** a newer commit cancels an older build that's still running.
- **Runner:** `macos-14` (Apple Silicon, arm64), with a 25-minute timeout.
- **Steps:**
  1. Check out the repo, then set up Python 3.14 and `python-tk` (needed for the app's control panel).
  2. Read the app version from `census_ui/__init__.py`.
  3. Create a venv and install the app plus PyInstaller and pytest.
  4. Provision the **agent's own Python interpreter** (`build_agent_python.py`, with a cached download). This is the Python the app's AI agent runs scripts with.
  5. **Run the tests.** All must pass before a build.
  6. Build the `.app` with PyInstaller (`census_ui.spec`), then the `.dmg` with `build_dmg.sh`.
  7. **Verify the bundle:**
     - the app and Info.plist are valid
     - the agent interpreter runs inside the app
     - **no real baseline (PHI) data is embedded**: the build fails if a `.dev-seeds` marker exists
     - the DMG has its first-launch panel
  8. Upload the DMG as a build artifact (kept 30 days), then delete old artifacts so only the newest 5 remain.

## `.env.example`
A template listing root-level keys: `LINEAR_API_KEY` and `CLICKUP_API_KEY`. You copy it to `.env`, which is gitignored, and fill in real values. Most projects have their own `.env.example` instead.

## `.gitignore`
Keeps out of git:
- machine junk: `.DS_Store`, `.idea/`, `node_modules/`, `__pycache__/`, `*.pyc`
- secrets: `.env`, `.claude.json`, `.claude/settings.local.json`, `mds/linear/linear-api-key.md`
- personal scratch: `local/`
- **census PHI:** `departments/sales/census_ingestion/assets/{inputs,outputs,intermediate}/`

Many modules add their own `.gitignore` for their raw or PHI folders.

## Conventions used everywhere

| Convention | Meaning |
|---|---|
| `AGENTS.md` in a folder | "Read this first", for agents and people. Some modules use `README.md` or lowercase `agents.md`. |
| `.agents/` folder | The project's memory: design, roadmap, decisions, learnings, milestones. Not run by anything. |
| `harness/` folder | The working kit: `workflows/` or `skills/` (procedures), `rules/` (what correct looks like), `tools/` (code). |
| `connectors/` | How a project reaches outside systems (Google, Intercom…). Never secrets. |
| `assets/`, `fixtures/` | Inputs and outputs. Real data is gitignored; tracked data is synthetic. |
| `VERSION` file | Semantic version (MAJOR.MINOR.PATCH) for a module. |
| Google Drive | Source of truth for briefs, docs and decisions. The repo holds code and agent notes. |
| `gws` | The Google Workspace CLI all Google access goes through. |
| Commit style | `type(department/module): description`, for example `feat(operations/unified_crm_cockpit): …`. Branches are named `<department>/<topic>`. |

---

# Part 2: `departments/ai_dept/`, the AI department

The AI department's own work: **how AI initiatives are chosen, how people and agents are set up, and the decks that explain it.**

| Module | Role in one line |
|---|---|
| `ai_initiative_shop/` | The process that takes a department pain → proof of concept → supported AI tool |
| `ai_org_deck/` | Exec decks: "Building an AI-Ready Organization" and "AI Across Zorro" |
| `ai_projects_setup/` | Setup for **agents**: the standard project shape + a Google identity per project |
| `ai_workstation_setup/` | Setup for **people**: role profiles, tool registry, Mac setup kit, verification, scheduler |
| `investor_materials/` | The AI story for investors and a slide map of the April 2026 investor deck |

How they connect:
- The shop defines the vocabulary: paths A–D, data levels, support tiers, and the PHI rule.
- The setup modules apply it.
- The **workspace kit** packs `workspace-base/`, the Google connector and the project standard into one installer.

---

## 2.1 `ai_initiative_shop/`

**Purpose:** a simple way for a department to submit a pain point, and a robust way to run the resulting tool, without each department reinventing ownership, support and data handling.
- People submit on their department's tab in the Google Doc "AI Initiative Candidates".
- This folder defines the process behind that doc.

**The one-line rule:** every initiative must answer *who owns it · what support tier · how sensitive the data is · what happens when it breaks · which system runs it*. If any answer is missing, it doesn't leave submission.

| File | What it is |
|---|---|
| `README.md` | Index and the one-line rule |
| `candidate-evaluation-criteria.md` | **The 5 scoring dimensions** (below) |
| `systems-paths.md` | **The 4 allowed ways to build** (below). Pick the cheapest one that meets the data and support needs |
| `lifecycle.md` | **The 5 phases:** Submission → Evaluation → POC → Scale & Release → Maintain & Support, each with a gate and an owner. Worked examples: Carrier Enrollment, Census Normalization, KB → Notion |
| `risks.md` | 10 risks (PHI exposure, shadow AI, POC graveyard, vendor lock-in, bus factor, quality drift…) and the phase that catches each |
| `initiatives/phi-ready-llm-access.md` | Brief: how sales reps can safely use AI on patient data through **AWS Bedrock**. Principle: **"refuse, don't fall back"** |
| `initiatives/phi-ready-llm-access-security-levels.html` | Visual of the credential security levels |
| `deck.md`, `build_deck.py`, `ai-initiative-shop-exec.pptx/.pdf` | Deck source, python-pptx builder, built 7-slide exec deck |

**The 5 evaluation dimensions:**

| Dimension | Options |
|---|---|
| Ownership | Department-owned · citizen-built in SaaS · co-owned · AI-Dept-owned · vendor-owned. Plus named Accountable / Builder / Support people |
| Support tier | T0 Experimental → T3 Mission-critical, with response times |
| Data exposure | Public · Internal · Confidential/PII · **Restricted/PHI** |
| Fallback | Manual · cached · failover · hard-stop |
| Systems path | One of A–D, below |

**The 4 systems paths:**

| Path | What it is |
|---|---|
| A | **Cassidy**: agentic workflow SaaS |
| B | **Claude hands-on**: Claude Code with written rules (most agentlab projects) |
| C | **Custom app**: real software (e.g. Census Raccoon, the CRM Cockpit) |
| D | **Google Apps Script / Gemini Gem** |

**Rules:**
- No phase can be skipped.
- PHI goes to Compliance for BAA confirmation **before** any POC.
- No runbook means no release.
- T0 tools are reviewed every quarter: retire or promote.

---

## 2.2 `ai_org_deck/`
Exec presentations, both credited to the AI Department.

| Path | What it is |
|---|---|
| `deck.md`, `build_deck.py`, `ai-org-deck-exec.pptx/.pdf` | "Building an AI-Ready Organization", 10 slides: agent loops, initiatives across the org, engineering examples (Figma → code, testing harness), managing AI across the org, the Initiative Shop |
| `overview/deck.md`, `overview/build_deck.py`, `overview/*.pptx/.pdf` | "AI Across Zorro", 7 slides for executives and non-engineering teams. Introduces 5 **Tracks**: Workflow Automation, Knowledge & Search, PHI / Systems-of-Record, Insight & Analytics, Creative & Content |

Rebuild with `/tmp/aisaip-venv/bin/python build_deck.py`. This is a Python venv with python-pptx.

---

## 2.3 `ai_projects_setup/`: setup for agents

**Purpose:** each agent project gets its **own Google service account** with the narrowest access, checkable by script. It is never a person's login.

**The 7 rules (`agents.md`):**
1. One service account per project.
2. Least privilege by **sharing specific folders**, not IAM roles or domain-wide delegation.
3. Secrets never go in the repo.
4. Every connector ships with a read-only verification script.
5. A connector is installed by the project's own agent, following a written workflow.
6. A connector installs into a project; it does not become one.
7. Granted access is recorded in a `verified tools` table, so agents never ask anyone for access.

### `connectors/google/`: Google access via `gws`
Two versions:

| | v1 (deprecated 2026-10-11) | **v2 (current)** |
|---|---|---|
| Where the code lives | A copy inside each project (`skill/`) | One shared copy per workspace (`workspace/` → `~/Workspaces/.shared/`) |
| Identity | Stamped into each project | `<project>/connectors/google/connector.conf` |
| Protection | A Claude "guard hook" script | The `gws` **wrapper** itself, which refuses forbidden commands |
| Credential | Key file | **Keyless** (preferred): your `oauth2l` sign-in mints a 15-minute token. Fallback: the key in the macOS Keychain |

| File / folder | What it is |
|---|---|
| `agents.md` | Design rationale, gotchas, list of provisioned accounts, change log |
| `playbook.md` | Admin checklist: provision → share folders (Viewer by default) → keyless grant → verify → store → rotate quarterly → revoke |
| `service_account.sh` | Admin CLI: `provision`, `rotate`, `store` (to 1Password), `install`, `verify`, `run`, `env` |
| `INSTALL.md`, `skill/`, `host_templates/` | **v1 (deprecated):** install workflow, per-project payload (shim, guard hook, identity check), files merged into a host project |
| `workspace/bin/gws` | **The v2 wrapper.** It finds the project you're in and runs gws as that project's account. **It allows** Sheets, Docs, Slides, Drive (no sharing changes) and read-only Calendar. **It refuses** Gmail, `auth` and everything else |
| `workspace/connectors/google/AGENTS.md` | Contract agents read: never ask for access, run gws from inside the project, `supportsAllDrives` rules, failure table |
| `workspace/connectors/google/setup-google.sh <project>` | Per-machine setup: installs gws/oauth2l, signs in or stores the key, verifies, records PASS |
| `workspace/connectors/google/templates/connector.conf` | Identity template: account name, `AUTH="impersonate"`, `SCOPES="drive"` |
| `test.sh`, `test_skill.sh`, `test_wrapper.sh` | Self-tests (69 offline wrapper checks, and others) |

**Provisioned accounts** (GCP project `internal-ai-systems`):
- v1: `enrollment-copilot-extension`, `renewals-ai-agent`
- v2 keyless: `careers-screening-agent`, `enrollment-dashboard-agent`, `help-center-sync-agent`, `ichra-objections-agent`

**Gotchas:**
- A "quota project" setting causes a 403 error. The fix is a dedicated gws config folder.
- Read-only access comes from the **Drive role (Viewer)**, not from scopes.
- Always pass `supportsAllDrives`.
- Service accounts have no My Drive, so they can only create files in shared drives.
- The Homebrew formula is `googleworkspace-cli`. The formula named `gws` is a different tool.

### `project_standard/`: the shape of every agent project
Builders copy `template/` to `~/Workspaces/<project>/`.

| Template path | Role |
|---|---|
| `AGENTS.md` | Project entry point: description, `## Team`, before-you-start steps, task types, hard rules, `## External boundary`, tools tables, **required signal** (🌻 + project emoji) |
| `HANDOFF.md` | Plain-language guide for the human owner |
| `.env.example` | Secrets as 1Password references only (`op://AI Builders/...`) |
| `connectors/AGENTS.md` | Index of connectors |
| `connectors/google/AGENTS.md`, `connector.conf` | The project's Google access map and identity |
| `harness/rules/` | Business rules for tasks |
| `harness/skills/` | Named procedures in plain Markdown |
| `harness/tools/` | Code |
| `harness/schedules/` | Recurring runs (installed with `lst`) |
| `harness/state/` | Machine-written JSON state between runs |
| `.agents/mission_background.md` | Why the project exists |
| `.agents/invariants_and_learnings.md` | Truths learned the hard way |
| `.agents/deliverables_and_roadmap.md` | What it delivers, what's next |
| `.agents/run_log.md` | Dated diary of runs |
| `assets/MANIFEST.md` | Index of per-run folders (`inputs/` immutable + `outputs/`) |

Rules: folder notes are always `AGENTS.md`, never README, and the shape is **provider-neutral** (nothing operational in `.claude/`).

---

## 2.4 `ai_workstation_setup/`: setup for people

**Purpose:** for any employee, answer: *what may I install, how do I connect it, what permissions should I hold, and how do we prove it?* It prevents both under-provisioning (a week lost) and over-provisioning (admin or PHI access nobody notices).

**One-line rule:** a role profile isn't done until a script or skill can **verify** it.

**Non-negotiables:**
- least privilege, with a written reason for each added scope
- never widen a live install to make a task pass
- verification is read-only
- never print secrets
- PHI goes through the approved channel or stops

Status:
- the `operator` profile is complete (verified 2026-07-28); the others are stubs
- workspace kit **v0.3.0**; `lst` **v0.2.0**

### `kits/workspace/`: one-script Mac setup for operators and builders
| Path | What it is |
|---|---|
| `AGENTS.md` | Full design: what each step installs, decisions, learnings, limits |
| `setup-workspace.sh` | Installer source. Flags: `--check`, `--dry-run`, `--workspace-only`, `--with-vscode`, `--yes`. Logs to `~/Library/Logs/zorro-workspace-setup.log` |
| `build.sh` | Bundles `workspace-base/` + the Google connector + the project standard into one file |
| `dist/setup-workspace.sh` | **The committed single file people actually run** (also on the AI Projects drive) |
| `test_kit.sh` | 70 checks |

**`workspace-base/`** becomes `~/Workspaces/`:

| Path | What it is |
|---|---|
| `AGENTS.md` | **Root rules for every session.** The user isn't an engineer; start replies with 🌻; ask which project; work only inside it; never edit `.shared/`. **External boundary:** never email, invite, message, share or publish to outsiders; if a task needs it, stop and post once in `#maya-ai-projects`. Plus 8 numbered project rules (draft never send, data-driven decisions, never delete Drive files you didn't create…) |
| `.claude/settings.json` | **Deny rules:** `gws auth`, the real gws binary, `oauth2l`, `security`, reading `.env` and credential folders, the Gmail/Drive/Calendar connectors. A start hook puts `.shared/bin` first on PATH |
| `.claude/commands/import-project.md` | The `/import-project` command |
| `.env` | **Backstop:** makes the real gws fail anywhere inside the workspace |
| `.gitignore` | Keeps data files, secrets and logs out of git |
| `.shared/bin/fill-env` | Reads `.env.example` → fetches from 1Password → stores in the **Keychain** |
| `.shared/bin/with-secrets` | Runs one command with the project's secrets |
| `.shared/bin/import-project` | Safely imports a project zip (never overwrites, strips secrets) |

Known limits:
- **Claude must be started at `~/Workspaces`**, not inside a project.
- The deny rules stop accidents, not a determined agent.
- The kit hasn't been tested on a truly fresh Mac yet.

### `patterns/`: how-to-connect principles
| File | What it is |
|---|---|
| `AGENTS.md` | Key ideas: "it works" proves enough access, not *correct* access, so test what should fail too |
| `connection-patterns.md` | 4 auth patterns: A per-user OAuth app (gws), B per-user OAuth via MCP or plugin (Slack, Linear, Figma), C personal token (GitHub PAT), D shared service credential (Bedrock key) |
| `credential-handling.md` | **"1Password for distribution, Keychain for use, never a dotfile."** Rotation, leak response |
| `permission-baselines.md` | Forbidden scopes (admin directory, reports, cloud-platform…), the verified 12-scope gws baseline |

### `profiles/`: access tiers
| Tier | Who | File | State |
|---|---|---|---|
| 1 user | Most staff, AI through a UI | `user.md/.json` | stub |
| 1 user-phi | Field sales / member data (PHI) | `user-phi.md/.json` | stub; blocked on choosing a credential model |
| 2 operator | Runs prepared agent projects | `operator.md/.json` | **complete** |
| 3 builder | Creates and shapes agent projects | `builder.md/.json` | stub |
| 4 engineer | Production code; only tier that merges and approves PRs | `engineer.md/.json` | stub; open PHI questions |

- `role-mapping.md/.json`: job title → profile (proposed).
- `AGENTS.md`: tiers, add-ons (`local-dev`, `design`, `google-personal`), the manifest schema.

Each profile is a **pair**: `.md` (the reasoning) + `.json` (the machine-readable install list).

### `scripts/`
| File | What it is |
|---|---|
| `install.sh` | Installs a profile: `--list`, `--profile`, `--title`, `--department`, `--dry-run`. **Refuses stub profiles** |
| `verify-gws-auth.sh` | Read-only check that your gws login has no forbidden admin scopes |

### `skills/`: Claude skills
| Skill | What it does |
|---|---|
| `setup-workstation/` | Walks a person through setup for their profile, then verifies |
| `verify-workstation-auth/` | Read-only audit of someone's auth against the baseline |
| `local-scheduled-task/` | **`lst`**: run any command or an unattended Claude session on a schedule on your Mac (below) |

**`lst` (local-scheduled-task):**
- Uses macOS **launchd** to open a terminal at set times.
- Verifies at 4 levels: L1 installed, L2 machine wakes, L3 fired on time, L4 task completed.
- Commands: `./lst selftest`, `catalog`, `install`, `verify`, `run`, `list`, `wake`, `show`, `uninstall`.
- Files:
  - `lib/`: Python and bash implementation (spec, registry, render, verify)
  - `templates/`: plist, launcher, runner, done-marker
  - `catalog/`: ready tasks: smoke-test, claude-smoke-test, **slack-briefing**
  - `tests/run-tests.sh`: 72 checks
- Projects keep their schedules in `harness/schedules/`.

### `templates/` and `tools/`
- `templates/role-profile-template.md`: the template for a new profile.
- `tools/`: **the approved-tool registry**, one file per tool:

| Folder | Tools |
|---|---|
| `base/` | homebrew, **claude-code** (CLI only; desktop and web disallowed), zed, 1password-cli, local-scheduled-task |
| `code/` | github (PAT must be approved by an org admin), lsp, frontend-design |
| `browser/` | chrome-devtools, playwright |
| `kb/` | **gws** (personal vs project-account modes), notion, oauth2l |
| `work/` | linear, **slack** (sees your DMs too), figma |
| `data/` | axiom (restricted: prod logs may hold PHI), psql, aws |
| `access/` | Grants to request: anthropic-seats, google-groups, internal-ai-systems, 1password-vaults, github-pat, linear-api-key |

**Standing rules:**
1. Claude Code CLI only.
2. Log in to the seat with email, not Google SSO.
3. Narrowest scopes; never `cloud-platform`.
4. **PHI never reaches a standard Claude subscription.**
5. Secrets go in the Keychain or 1Password, never a shell profile.

---

## 2.5 `investor_materials/`
| File | What it is |
|---|---|
| `Investor Deck April 2026.pptx` | The 34-slide investor deck |
| `ai_pillars.md` | **The source AI narrative:** P1 Product AI, P2 Operations AI, P3 Engineering AI. Each has a marketing line, concrete achievements, and technical depth |
| `presentation_slides.md` | Slide-by-slide map of where to add AI content |
| `local.md` | Empty |

Rules:
- Claim only shipped or in-progress work.
- Name what the AI does, not just "AI".
- Claim the moat from data and workflow, not the model.

---

# Part 3: `departments/sales/`

Two modules, the largest area of the repo (~700 files):

| Module | What it does | Users |
|---|---|---|
| `census_ingestion/` (**Census Raccoon**) | Turns brokers' messy census files into Zorro's standard census spreadsheet | Account Executives (AEs) |
| `proposals/` (**Proposal Studio**) | Generates on-brand interactive HTML and PDF sales proposals | AEs → prospects |

---

## 3.1 `census_ingestion/`: Census Raccoon

### Purpose
- **Input:** broker census files (employee rosters and rate sheets, as xlsx, csv or pdf).
- **Output:** a `normalized.xlsx` with an **Employees** sheet and a **Plans** sheet, ready to load into **Tornado**, Zorro's quoting system.
- It started as a Claude Code harness. It's now also a **local web app** (FastAPI) using the Claude Agent SDK on **AWS Bedrock**, shipped as a **macOS app (.dmg)**.

### Status
- App v0.2.0. Milestone M1 "UI Workflow Control" (owner nadav.k).
- Steps profile → extract → normalize → validate → output work in the app. Trace and learn don't yet.
- Later milestones: M2 hosted multi-AE server, M3 Tornado upload. Both deferred.
- Releases are announced in `#contact-census-raccoon-mvp` and delivered via the Drive folder "Census Raccoon Releases for AEs".

### Folder map
| Path | What it is |
|---|---|
| `AGENTS.md` | Router: Drive layout, **strict PHI policy**, obfuscation rule, no stray docs, PHI pre-flight gate |
| `.gitignore` | Ignores everything under `asset_processing/` and `baseline_eval/` except `.gitkeep` |
| `requirements.txt` | Harness Python packages (openpyxl, PyYAML, Faker) |
| `reports/tool_usage_stats.md` | Log of tool runs |
| **`.agents/`** | Planning |
| `roadmap.md` | Phases P0–P6, constraints, distribution stages, open bugs (`BUG-XXX`) |
| `milestones/M1-ui-workflow-control.md` | M1 goals, user journey, sub-phases |
| `industry_ideation.md` | Prior-art survey; decision on service account + 1Password key |
| `distributions.md` | Dev mode vs bundled app differences |
| `tech_bugs_and_learnings.md` | ~1,100-line post-mortem log (Bedrock auth pitfalls, BUG-001…) |
| `architecture.html`, `prime_html/index.html` | Visual architecture and overview pages |
| **`asset_processing/`** | **Live enrollment work (PHI, gitignored).** One folder per census: `1_raw_inputs/` → `2_intermediates/` → `3_results/` |
| **`baseline_eval/`** | **Test set (PHI, gitignored):** `example1`–`example4` (Employers A–D) with `4_expected/` correct answers. Real files live in Drive |
| **`census_ui/`** | **The app** (below) |
| **`harness/`** | **The core pipeline** (below) |

### `census_ui/`: the app
| Path | What it is |
|---|---|
| `pyproject.toml` | Python package `census-ui`: fastapi, uvicorn, jinja2, keyring, boto3, **claude-agent-sdk** |
| `run.sh` | Dev launcher on port 5814 (`--watch` restarts on change). The bundled app uses 5813 |
| `build_agent_python.py` | Downloads a pinned Python 3.14 the app's agent uses to run scripts |
| `census_ui.spec` | PyInstaller recipe for the `.app`: bundles the harness, Claude CLI, templates and fixtures. Stamps the version |
| `build_dmg.sh`, `build_dmg_agreement.py` | Builds the `.dmg` with a first-launch notice. Tags `__WITH_PHI` if real data got embedded |
| `README.md`, `how_to_run.md` | Admin one-time setup (Bedrock model access, key into 1Password) + per-AE install |
| `docs/` | `AE_QUICKSTART.md`, `models_runbook.md`, `releasing.md`, first-launch text |
| `fixtures/baselines/example5_synthetic_micro/` | **Synthetic** test census with deliberate defects (safe to commit) |
| `tests/` | 21 pytest files, ~305 tests |
| `census_ui/` (package) | The app code (below) |

**App code (`census_ui/census_ui/`):**
- **Core:**
  - `__main__.py`: starts the server and opens the browser
  - `app.py`
  - `config.py`: models: default Claude Opus 5 on Bedrock, also Sonnet 5 and Haiku 4.5. Workspace: `~/Library/Application Support/Census Raccoon`
  - `key_store.py`: Bedrock key in the Keychain
  - `preflight.py`, `panel.py` (small Tk control window), `upgrade.py`
- **`llm/`:** Agent SDK client, step prompts (`step1_profile.md`, `step2_extract.md`, `step5_output.md`), and narration that turns tool calls into plain English.
- **`orchestrator/`:**
  - `phi_gate.py`: blocks if there's no valid Bedrock key
  - `pipeline.py`, `steps.py`, `code_steps.py`: deterministic normalize and validate, plus scoring
  - `gate_spec.py`, `gate_arbiter.py`: decision gates; auto-answers only with a real citation
  - `tool_sandbox.py`: the agent can't leave its enrollment folder
  - `autorun.py`: unattended runs
  - `candidate_mappings.py`, `learnings.py`
- **`routes/`:** setup (paste key), enrollment, step, stream (live updates), checkpoint (approve, adjust, reject), autorun, monitor, promote-mappings, learnings, admin.
- **`admin/`** (engineers only, not shipped):
  - `provision_bedrock_key.sh`: creates the AWS IAM user, log group and key
  - `publish_dmg.sh`: refuses PHI builds
- **`templates/`, `static/`:** web pages and CSS.

### `harness/`: the pipeline (also shipped inside the app)
| Path | What it is |
|---|---|
| `agents.md` | Index, PHI gate, pipeline summary |
| `design/design.md`, `approach_research.md` | Why the pipeline is shaped this way; confidence thresholds |
| `rules/normalized_form.md` | **The output spec:** 13 Employee columns (ID, dependent type, coverage tier EE/ES/EC/FA, DOB MM/DD/YYYY, 5-digit ZIP…) |
| `rules/normalization_mapping.md` | Messy value → standard value, per field |
| `rules/decision_gates.md` | **When the agent must ask a human:** participation, multi-sheet, plan selection, household boundaries, date formats, gap thresholds… |
| `rules/examples.md` | History of each example |
| `rules/templates/` | Report templates + **`zorro_census_template.xlsx` (the authoritative output format)** |
| `tools/profiler/` | Detects layout (tall, wide, far-right-wide), header rows, key columns |
| `tools/extractors/generic_extractor.py` | Config-driven extraction → `persons.csv`, `plans.csv` |
| `tools/normalizers/canonical_normalizer.py` | Intermediate CSV → `normalized.xlsx` + gaps report |
| `tools/diff_checker/` | Compares output vs the expected answer; scores 90+ Excellent … <50 Poor |
| `tools/baseline_report_generator/` | Combined HTML report across examples |
| `workflows/run_ingestion.md` | **Entry prompt for one enrollment** |
| `workflows/evaluate_baseline.md` | Run all examples, diff, score, improve rules |

### How it works
1. **PROFILE** (AI): understands the file → `input_profile.md`. **Human review #1.**
2. **EXTRACT + MAP** (AI): produces `persons.csv` and `plans.csv`.
3. **NORMALIZE** (code): produces `normalized.xlsx`.
4. **VALIDATE** (code): checks the output; the AI fixes soft failures, up to 2 attempts.
5. **OUTPUT** (AI): writes `result.md`. **Human review #2.**

AI handles messy understanding; code handles exact formatting.

**AE use:**
1. Install the DMG.
2. Paste the Bedrock key from 1Password once.
3. Create an enrollment and drop in files.
4. Run the steps (or autorun) and answer gate questions.
5. Download the result.

### Key commands
```bash
# Dev
cd departments/sales/census_ingestion/census_ui
python3 -m venv .venv && .venv/bin/pip install -e .
./run.sh --watch        # http://127.0.0.1:5814
pytest

# Build
python3 build_agent_python.py
CENSUS_UI_NO_DEV_SEEDS=1 .venv/bin/pyinstaller --clean --noconfirm census_ui.spec
./build_dmg.sh
```

### Rules
- **Strict PHI:** everything in `asset_processing/` and `baseline_eval/` is PHI, even `.md` files. Only `.gitkeep` is tracked.
- **PHI pre-flight gate:** only AWS Bedrock (or Claude Enterprise with a BAA) may read real census content.
- **Obfuscation:** tracked files use only fake names, `XXX-XX-1234` and Employers A–D. Two past leaks required history rewrites.
- **No stray docs:** session notes go to `/tmp/`, not the repo.
- **Per-AE isolation:** each AE sees only their own census data.
- **Builds containing real data** are tagged `__WITH_PHI` and must never be distributed.

### The CI workflow
`.github/workflows/build-census-raccoon-bundle.yml`, described in Part 1:
- It builds and tests the app on each relevant push to `main`.
- It fails if PHI got embedded.
- It keeps the last 5 DMGs as artifacts.

---

## 3.2 `proposals/`: Proposal Studio

### Purpose
Turns the design inputs into Zorro sales proposals:
- inputs: Figma designs, the design spec, the marketing outline (Google Doc) and Slack feedback
- outputs: a dark-themed **interactive HTML** deck and a light **printable PDF**

AEs use an **offline single-file "Proposal Studio"** in Chrome:
1. Load a Tornado quote export.
2. Pick their identity, choose designs, confirm fees, edit copy.
3. Download the HTML and PDF.

### Status
- Template edition **v0.8**; factory **v0.8.31-beta** (published 2026-10-12, 485/485 tests passing).
- Built to replace **Qwilr**: V1 was due Aug 7, 2026, and the Qwilr contract ended Aug 15, 2026.
- Slack: `#ai-project-newgen-proposals`. Owner: Nadav. Design and brand people are listed in `requirements.md`.
- Change requests are tracked in `1_sources/change_requests.md`.

### The strict development flow
1. Explore ideas in `2_harness/experiments/`. Nothing there ships.
2. Put decisions into the **template** (`2_harness/templates/proposal/v0.8/`).
3. **Regenerate** outputs with the CLI. **Never hand-edit `3_publications/`.**
4. Confirm the version and scope with the owner before editing. Re-render and diff after every change.

### Folder map
| Path | What it is |
|---|---|
| `AGENTS.md` | Router: the strict flow, sources, rule cascade, rules |
| `.gitignore` | Real prospect publications are ignored (PII) |
| **`.agents/`** | Project memory |
| `roadmap/` | `plan.md` (4 lanes: Presentation, Capability, Correctness, Distribution), `requirements.md`, `decisions.md`, `deliverables.md` (shipped work + Drive release log) |
| `knowledge/register.md` | Invariants INV-01…13, bugs BUG-01…60+, learnings |
| `architecture/` | Solution concept, flow, hosting plans, `topics/` (fees, Tornado contract, studio editor, responsiveness…) |
| `workflows/` | Runbooks: responsiveness testing, manual fee reconciliation vs Tornado |
| `evidence/` | Dated proof folders (screenshots) |
| **`1_sources/`** | **Pointers** to the source inputs |
| `AGENTS.md` | Drive layout, how to fetch with `gws`, Google Docs API lessons |
| `requirements_and_product_spec.md`, `marketing_outline.md`, `figma.md`, `slack.md`, `ae_directory.md` | Mirrors and pointers |
| `change_requests.md` | **The change-request register** (CPY/CNT/FNC/BUG codes) |
| `outline/` | Outline template + dated pulls |
| `tools/gdoc_table/gdoc_table.py` | CLI to read and edit Google Doc tables (dry-run unless `--go`) |
| **`2_harness/`** | **The generator** |
| `experiments/` | ~35 throwaway design option pages; `hosting_poc/` (prototype for hosting proposals at a secure link) |
| `rules/` | **Rule cascade.** Base: `design_system.md`, `language_book.md`, `interactivity.md`, `figures_and_terminology.md`. Screen-only: `responsiveness.md`. Overrides: `output_profiles.md` (HTML vs PDF), `variants.md` (champion vs executive) |
| `templates/proposal/v0.8/` | **The template:** HTML and PDF templates, input schema, section map, sample data, fonts, icons, carrier logos |
| `tools/` | Agent notes (`build_proposal.md`, `render_html.md`, `render_pdf.md`), `review_pdf.sh`, `proposal_paths.json` |
| `tools/studio/` | **The TypeScript pipeline** (below) |
| **`3_deliverables/`** | What AEs get |
| `studio/proposal_studio_v0.8.31-beta.html` | **The Studio "factory" file** AEs download from Drive |
| `tornado_dumps/` | Synthetic Tornado quote exports only |
| `AE_QUICKSTART.md` | Guide text (the shipped guide is built into the Studio) |
| **`3_publications/`** | Generated proposals per prospect. Only `tornado_sample_prospect/` (synthetic) is committed |

**`2_harness/tools/studio/`** is numbered pipeline stages in `src/`:
1. `1_inputs`: read the Tornado quote
2. `2_plan_pricing`
3. `3_fees`: one fee ledger
4. `4_composition`: text, timeline, logos
5. `5_rendering`: template, PDF via headless Chrome
6. `6_delivery`

It also contains `studio/` (the AE app), `command_line/`, `checks/`, `dist/` (compiled; committed) and `test/` (goldens, fee oracle, end-to-end AE journey).

### Key commands (inside `2_harness/tools/studio/`)
```bash
npm install && npm run build
npm test                                   # PE_REQUIRE_FULL=1 npm test for releases
npm run proposal -- read-quote <dump.json> --out data.json
npm run proposal -- render <proposal.json> --data <data.json> --out 3_publications --formats html,pdf
npm run proposal -- build-studio --version v0.8
```

### Rules
- **Template first**; outputs are always regenerated.
- **Brand:** sanctioned icons only. HTML is dark; the PDF is light.
- **PII:** real Tornado dumps and real prospect proposals are never committed. The adapter never copies employee rows into output (INV-02).
- **One fee ledger** for every number, and adjacent figures must agree.
- Bump `factory_patch` and `factory_build` together on each Studio change.

---

# Part 4: `departments/operations/`

| Module | What it is | Stack |
|---|---|---|
| `carrier_enrollment/` | **Zorro Enrollment Copilot**: Chrome extension that pre-fills carrier enrollment portals with a human in the loop | TypeScript/JS, Playwright, Python, Go |
| `unified_crm_cockpit/` | **"Zong"**: an offline HTML dashboard over Intercom data for customer teams | TypeScript (Node 24) |
| `customer_teams/` | Copies of 4 agent projects run by customer teams in their own workspaces | Markdown rules, Python, `gws`, Slack |

---

## 4.1 `carrier_enrollment/`: Enrollment Copilot

### Purpose
Ops staff re-type member data into carrier websites to enroll members. The Copilot:
- lets the operator use their own browser on the carrier site
- **pre-fills fields** from Zorro data, so the operator reviews instead of typing patient data
- verifies and records the result

It is **human-supervised by default**. Full automation is an opt-in per carrier.

### Status
- POC. Extension **v0.6.1**.
- First real carrier: **Capital BlueCross (CBC)**. Its recipe is still being built; the mock version is approved.
- The production data pull works but isn't cleared for real enrollments.
- Main customer: Bradley Howell (Ops). Priorities are tracked in a Slack canvas.

### Folder map
| Path | What it is |
|---|---|
| `AGENTS.md` | Mission, Drive links (project doc, **Carrier Table – Customer Ops** sheet = carrier source of truth), conventions |
| **`.agents/`** | Planning |
| `roadmap.md` | **Start here.** 3 lanes: Recipe CI/CD, Recordings & Submission, Distribution |
| `distribution/` | How to ship the extension: Chrome Web Store Private (fallback), self-hosted `.crx` (goal), unpacked + self-updater (near-term); draft privacy policy |
| `foundations/` | Original problem definition, POC plan, v0 scope (historical) |
| `industry_ideation/` | Survey of 23 browser-automation vendors and bot-detection techniques |
| `prime_html/` | HTML overview dashboard |
| `recipes/` | How carrier "recipes" are learned; priority carrier list; application-info scope |
| `reference/architecture.md` | Solution shape, data contracts |
| `reference/invariants_learnings_and_bugs.md` | **Invariants I1–I9.** Read before any design decision |
| `submission/` | Display hints, session model, repeating dependents |
| **`harness/`** | The code |
| `Makefile`, `dev.sh` | All commands |
| `agent_workflows/` | Agent-led runbook (stops before submit) |
| `carrier_registry/configs/<carrier>/` | **Recipes:** `flow.json` (pages, fields, buttons), `metadata.json` (from the Ops sheet), `status.json` (state and score). Carriers: aspirus, capital_bluecross, mock_capital_bluecross |
| `carrier_registry/drafts/` | Auto-drafted skeletons for 10 more carriers (no fields yet) |
| `carrier_registry/provider/`, `server/` | Recipe loading and validation; API stubs (a future `deployable` app) |
| `carrier_registry/tools/*.py` | Pull the Ops sheet via `gws` into JSON |
| `cli/` | v0 CLI: `probe` (AI-assisted page capture via Bedrock), `fill` |
| `extension/` | **The Chrome extension:** service worker, **sidebar** UI (matching, fill guards, recipe editor, recorder, Drive upload, sessions, dependents, SEP reasoning), content scripts |
| `fill-core/` | Fill engine (pure TypeScript, no dependencies) |
| `fixtures/` | **Synthetic** test members (invalid SSNs `000000XXXX`, `.test` emails, `555` phones) |
| `mock_carrier/` | Local fake CBC website on 127.0.0.1:8080 |
| `tests/` | Node test scripts |
| `tools/` | Recipe integrity checker, status scoring, self-replay (mock and real), live probe, auto-login (1Password), recordings tools |
| `updater/` | Self-update helper (Go; partly stubbed) |
| `walk_reports/` | CBC acceptance walk templates |

### How it works
1. **Recipes:** the Ops sheet becomes JSON; each carrier's `flow.json` is written or learned from recordings.
2. **Build:** `make build` bundles the recipes into the extension.
3. **Operate:**
   - The operator logs in **by hand** in a dedicated Chrome profile.
   - The sidebar recognizes the page and loads the member record.
   - The operator clicks Fill, reviews, and submits.
4. **Learn and verify:** recordings go to Drive → integrity check → replay against the mock or the real site → status score.
5. **Distribute:** `make package` produces a zip, uploaded to the shared Drive dev folder.

### Key commands (inside `harness/`)
```bash
make chrome          # isolated Chrome profile with debugging
make build | make watch | make dev-all | make mock
make test | make typecheck
make recipe-integrity-check CARRIER=…
make recipe-status CARRIER=…
make self-replay
make package
```

### Rules
- **PHI:** filling is deterministic, with **no AI at fill time**. Only Bedrock paths may reason over PHI. Bank data is memory-only and never logged.
- **The harness never holds carrier passwords.** Login is manual.
- **Never defeat captchas.** Be detection-aware, not evasive.
- **Never tune recipes against the mock**; the real site wins.
- **Fixtures are always fake.** Use real keyboard events.
- Recording uploads currently contain PHI (internal only). A review gate must return before wider release.

---

## 4.2 `unified_crm_cockpit/`: "Zong"

### Purpose
One dashboard for all customer teams (CREM, Support, Payments). It shows client issues and metrics across email, phone and chat, from the company, broker, carrier and employee perspectives, plus AI summaries and CSM quality checks.

**Phase 1:** a single offline HTML file built from Intercom data. No server, no login.

### Status
- **v0.14.0** (rules model v1.6.0). Most active project.
- Source of truth: the Google Doc "Unified Customer Relations Cockpit".
  - Lane A: the Cockpit.
  - Lane B: CSM Ops QA (spec by Amy).
  - Gap Report tab. **The Doc wins over the repo.**
- D1–D2 merged, D3 in review, D4 is a review with the Ops managers.

### Folder map
| Path | What it is |
|---|---|
| `AGENTS.md` | Router: layout, stack, versioning, publishing, PHI rules |
| `VERSION`, `package.json`, `tsconfig.json` | Version (single source); npm scripts |
| **`.agents/`** | `requirements.md` (IDs A-…, B-…), `layout_sketch.md`, `survey_intercom_60d.md`, `deliverable_roadmap.md` (D1–D9, releases), `gap_report.md` (GR-1…29, published to the Doc), `model_flow.md`, `state_issue_mapping.md`, `response_gap.md`, `client_reports.md` |
| **`fixtures/`** | `synthetic.json` (tracked fake data); `scrubbed/` and `raw/` are **gitignored** (raw = PHI) |
| **`integrations/intercom/`** | Intercom access notes; `data_taxonomy/` (entities, fields, joins, gaps) |
| **`integrations/llm/`** | The AI layer rules + versioned `prompts/` (ticket summary, gist, issue classify) |
| **`prototype/`** | `cockpit.html` (synthetic build, tracked); `pack.sh` (full pipeline); real builds gitignored |
| **`src/app/`** | The page: one module per tab: today, cockpit, tree, tickets, issues, heatmap, ops QA, reports, taxonomy, lineage, gaps, techflow |
| **`src/build/`** | `build.ts`: one self-contained HTML |
| **`src/model/`** | **All business rules**: rule registry (id, version, owner, status), data registry, typed conditions, rules (carrier, coverage, flags, health, service, signals, state tags…), CHANGELOG, generated model files |
| **`src/pipeline/`** | Intercom → data: `dump` (full read-only pull, PHI) → `analyze` → `enrich` (service levels) → `adapt` (scrubbed or real) → optional `annotate` and `issue_classify` (AI) |
| **`src/publish/`** | `gws.ts`, `publish.ts` (to Drive), `md_to_doc_tab.ts` (Gap Report → Doc), `export_reports.ts` (PDFs per admin or broker) |

### How it works
```
Intercom ──dump──► raw (PHI, local) ─► analyze ─► enrich ─► adapt ─► fixture JSON
                                                         (AI annotate, optional)
fixture JSON ─► build ─► one offline HTML ─► publish ─► restricted Drive folder
```
- **Every decision is a versioned rule in `src/model`.** The page and pipeline only ask the model.
- **AI never decides.** AI text is marked ✦ and never feeds counts or flags.

### Key commands
```bash
npm run typecheck && npm test && npm run model
npm run synth && npm run build        # synthetic build → prototype/cockpit.html
npm run pack                           # full real pipeline (needs Intercom token in .env)
node src/publish/publish.ts prototype/dist/<file>.html --dry-run
```

### Rules
- **Raw Intercom data never enters git.** Tracked files use fake names.
- **Approved AI route:** only Claude in the owner's interactive Claude Code session (`annotate.ts --backend session`). Unattended AI needs its own approved path.
- **Changing a rule:** bump the rule's version and `MODEL_VERSION`, add a CHANGELOG line, update its test and its Lineage entry. `npm run model` enforces this.
- **Bump `VERSION`** in the same commit.
- **Publishing refuses** if the Drive folder is shared too widely. Old builds are archived, never deleted.
- No README files; record unknowns as `GAP:` / `UNVERIFIED`.

---

## 4.3 `customer_teams/`: agent projects run by customer teams

### What this folder is
**Snapshots ("checkpoints")** of 4 agent projects that ops staff run in their own `~/Workspaces`.
- They were handed over by Maya Perl in Oct 2026 and standardized to the project standard.
- **Don't edit them here.** Change the working copy, then checkpoint again.
- Folder names are kebab-case because they're the key for `setup-google.sh` and the Keychain.

### Shape of each project
Every project follows the project standard (Part 2.3):
- `AGENTS.md`: Team, rules, external boundary, 🌻 signal
- `HANDOFF.md`
- `.env.example` (some projects)
- `.agents/`
- `assets/`: dated runs
- `connectors/google`: own keyless service account
- `connectors/slack`: the operator's own Slack
- `harness/rules/NN-*.md`

### The 4 projects
| Project | What it does | Owner | Key details |
|---|---|---|---|
| **enrollment-dashboard** | Builds the leadership **Enrollment Dashboard** HTML from Metabase exports and overwrites one permanent Drive file | Bradley Howell | Current v11. Python/pandas stages (`run_all.sh`, `build_html.py`). Binding metric formulas in rule 04. **Publishing needs owner approval**; no automatic Slack posts; no employee-level data on the page |
| **help-center-sync** | For a product or process change, finds every **help-center article and SOP that is now stale** and proposes exact edits | Lerone Lessner | Outputs a proposal Doc + tracker sheet. Finding types: Remove, Rewrite, Screenshots only, No change, New needed, Flag, Adjacent. **Never edit master docs; never publish to Intercom** |
| **ichra-takeover-objections** | Reviewed answers and transition decks for prospects **switching to Zorro from another ICHRA administrator** | Ann Marie G (Ops reviewer: Bradley Howell) | Factual guardrails, objection bank, deck template. Ops and the owner both approve; **the AE sends**. Never invent facts |
| **cs-implementation-recruiting** | **Screens CS/Implementation job applicants** and keeps the interview tracker current | Ann Marie G + Bradley Howell | Scoring lenses (domain fit, excellence, benefits experience; US-based required). Scheduled 3×/day via `lst`. **Never contact candidates**; people make every hiring decision |

### Kept local only (never in git)
- **All projects:** secrets, `.env`, spreadsheets
- **recruiting:** resumes, run log
- **dashboard:** enrollment data and ARR model
- **objections:** the prospect's materials
- **help-center:** one screenshot with personal data

### Open items
- **The enrollment dashboard input includes PHI.** The proposed fix is to aggregate in Metabase; it needs a compliance decision.
- Create the 1Password items the projects reference.
- Move outputs to the shared drive "AI Projects – Limited Access".

---

# Part 5: `departments/compliance/`, the AI Policy module

## Purpose
Drafts, reviews and publishes Zorro's **AI and data-governance policies**. Zorro is a HIPAA-regulated health insurer that handles PHI.
- Policies are written in Markdown here, then converted into branded **Google Docs**.
- They are tracked in a Google Sheet register called **Policy Management**.

## Status
- **Nothing is approved or published yet.** `publications/` is empty and the Decision Log has only its header row.
- **7 policies are drafted (v0.x):**

| ID | Policy | Owner |
|---|---|---|
| POL-001 | Data Classification | Chief Compliance Officer |
| POL-002 | Data Usage | Chief Compliance Officer |
| POL-003 | AI Data Usage with LLM | VP AI Engineering |
| POL-004 | AI Systems Integrations | VP AI Engineering |
| POL-005 | Google Drive | CISO |
| POL-006 | Slack | Operations Manager |
| POL-007 | AI Foundations (entry point) | Chief Compliance Officer |

- The older `policy/` set (Workforce, Builder, one-pager, Appendix 5) is **draft v1.0**. It is still the fuller text until the new articles are approved.
- **Six open questions (q1–q6)** block final sign-off.
- The Member-Facing AI Policy is deferred to v1.5. **Member-facing AI is prohibited in v1.**

## Folder and file map

| Path | What it is |
|---|---|
| `AGENTS.md` | **The operating contract** (11 rules, below), the Drive layout and IDs, terminology |
| **`management/`** | Working area for drafts and the register mirror |
| `management/README.md` | The two sheet tabs: **Policy Management** (current state, edited in place) and **Decision Log** (history, append-only) |
| `management/decision_log.csv` | Read-only snapshot of the Decision Log. **Never edit it**; refresh it with `--pull` |
| `management/ramp_up_publish_history.csv` | Record of ~40 draft re-publishes (Sep 2026). Not the Decision Log |
| `management/comments/` | **Harvested Google Doc comment threads** (e.g. 8 Slack-policy files). Permanent; never delete |
| `management/drafts/INVENTORY.md` | Status table of all drafts, drafting order, repo → Drive mapping |
| `management/drafts/foundations/ai_foundations.md` | POL-007: where models may run, access levels, usage classes, AI incidents, exceptions |
| `management/drafts/general/data_classification.md` | POL-001: the 5 data classes, incident severity |
| `management/drafts/general/data_usage.md` | POL-002: approved tools, PHI handling, human review, new-tool approval |
| `management/drafts/ai/data_usage_with_llm.md` | POL-003: what may enter an AI's context, prompt injection, retention, provider assessment |
| `management/drafts/ai/systems_integrations.md` | POL-004: rules for builders: runtimes, agents and tools, registries, supply chain, secrets, evaluation |
| `management/drafts/systems/google_drive.md` | POL-005: what may live in Drive, AI features, admin settings baseline |
| `management/drafts/systems/slack.md` | POL-006: what may be posted (members by ID, no claim screenshots), channels, apps |
| **`policy/`** (legacy, draft v1.0) | Uses the older `[MUST]/[SHOULD]` markers and T/L/A codes |
| `policy/workforce_ai_policy.md` | Full staff policy §§1–16 plus appendices (classification, permission matrix, approved tools, exceptions) |
| `policy/builder_ai_policy.md` | Engineering policy §§1–19: runtimes, intake gate, evaluation, agents, MCP, supply chain, logging, secrets, incidents |
| `policy/workforce_ai_policy_one_pager.md` | Plain-language summary: "Never put PHI into an AI tool that isn't on the approved list" |
| `policy/appendix_5_standards_crosswalk.md` | Maps each clause to HIPAA, NIST AI RMF, ISO 42001, NAIC, the EU AI Act and state laws |
| **`process/`** | Planning and research |
| `process/v1_policy_plan.md` | Plan: anchor standards, 10 "red line" criteria, delivery order |
| `process/dirty_initial_research.md` | Raw early research notes |
| `process/source_references.md` | Research sources (URLs need re-sourcing) |
| `process/open_questions/q1…q6` | **Blocking questions:** q1 operating states and state laws · q2 EU exposure · q3 existing governance committee · q4 what AI vendor BAAs cover · q5 existing Acceptable Use Policy · q6 federal preemption wording |
| **`publications/`** | Empty by design until something is approved. README explains the approval steps |
| **`templates/`** | Policy templates (`VERSION` = 2.0.0) |
| `templates/v2.0.0/policy_article.md` | **The current article template**: Overview, Scope, Definitions, Classification, Systems-only sections, Prohibitions, Exceptions, Incidents |
| `templates/v2.0.0/shared/clause_conventions.md` | Binding clause rules: Required/Never/Recommended markers, numbering, standard references |
| `templates/v2.0.0/shared/regulatory_alignment.md` | The §1 alignment clause copied into every article (pending q6) |
| `templates/v2.0.0/zorro_policy_template.docx` | Generated Word styling for conversion. Do not hand-edit |
| `templates/v1.0.0/` | Previous template set, kept because older Docs were built from it |
| `templates/examples/` | A real published Zorro policy PDF (target shape), the brand template docx, an older HR template |
| **`tools/`** | Scripts |
| `tools/push_to_drive.py` | **The publishing tool** (below) |
| `tools/build_reference_docx.py` | Builds the branded Word reference template from the brand docx |

## The rules (`compliance/AGENTS.md`)
1. **The Policy Management sheet is the source of truth** for version, status, owner and dates. Never hardcode them in a policy.
2. **The Decision Log holds approved versions only and is append-only.** Never edit or delete a row.
3. **Status decides where a Doc lands.** Not approved means Management › Drafts. Never move Docs by hand.
4. **One draft Doc per policy, for its whole life.** It is named `[POL-00n] <Policy>`, with no version in the name. It is never recreated, so its comments and history survive. *This was broken once (Slack, 27–28 Sep 2026) and can't be undone.*
5. **Never publish a version the register doesn't know.**
6. **Template versions are never overwritten.** A change means a new `vN.N.N/` folder.
7. **Comments are harvested before any replacing write.**
8. **Clause markers are `Required` / `Never` / `Recommended`,** each with a standard reference.
9. **Classes are named, never coded:**
   - data: Public · Internal · Confidential · Sensitive · Restricted
   - surfaces: Isolated · In-VPC · Covered · Partial · Unprotected
   - people: General · Trained · Builder · Governance
10. **Metadata stays out of the policy body.** The body carries only Policy ID, Effective Date, Applies To and Owner.
11. **Stay inside `departments/compliance/`.**

Drive access uses `gws` with the operator's own Google login; there is no service account. The Drive layout is `Policies/` → `Publications/`, `Archive/`, `Management/` (with the sheet and `Drafts/`).

## Policy lifecycle
1. **Draft** the Markdown from the v2 template, including a `**Policy ID:** POL-00n` line.
2. **Register** the policy in the Policy Management sheet: version, date, status, template version.
3. **Dry run:**
   ```bash
   python3 tools/push_to_drive.py <file.md>
   ```
   This shows what would happen and changes nothing.
4. **Push:**
   ```bash
   python3 tools/push_to_drive.py <file.md> --push
   ```
   - It converts the file with pandoc and updates the single draft Doc in place.
   - It first harvests comments, and **blocks (exit 2) if any are unresolved**.
   - Once those comments are folded into the Markdown, re-run with `--comments-carried`.
5. **Review:** reviewers comment on the draft Doc, and you fold the comments into the Markdown.
6. **Approve:** the approver fills in the Decision Log fields, and the status becomes `Approved` at `v1.0.0`. v0.x means draft; v1.0.0+ means approved.
7. **Publish:**
   ```bash
   python3 tools/push_to_drive.py <file.md> --push --publish
   ```
   - It makes a **read-only copy** in Publications named `[POL-00n] <Title> vX.Y.Z (date)` and appends one Decision Log row.
   - It refuses if the policy isn't approved or that version already exists.
8. **Refresh the local mirror:**
   ```bash
   python3 tools/push_to_drive.py --pull
   ```

**Versions:** major = new approved edition · minor = substantive change (needs re-approval) · patch = typo only.

## Known inconsistencies
- The docs mention `--log` and `--replace` flags that don't exist. The real flags are `--push`, `--publish`, `--comments-carried` and `--pull`.
- Some docstrings point to `publications/<group>/` paths; the drafts live in `management/drafts/`.
- `AGENTS.md` says Arial font while the tool uses Roboto. Its status text is stale.
- The legacy `policy/` files still use the retired codes.

---

# Part 6: `departments/general_admin/`

## Systems Manager (`departments/general_admin/systems_manager/`)

### Purpose
An internal web app that **replaces scattered Google Sheets** (vendor list, per-vendor cards, data-usage table) with one linked database. Anyone should be able to answer these in seconds:
- "Who owns X?"
- "What data does Y touch?"
- "Which systems send data to AI?"

### Status
The PRD is by Nadav K, dated 2026-04-05, status Draft.

| Milestone | Scope | Status |
|---|---|---|
| M0 | Google sign-in | Done |
| M1 | Database schema + seed data | Done |
| M2 | Vendor/system create/edit + UI | Done |
| M3 | "My Department" page | Done |
| M4 | "My Vendors" page | Done |
| M5 | Global dashboard | Next (pages already exist) |
| M6 | Integration map | Not started (a page exists) |
| M7 | Import from Sheets | Not started |

It runs **locally only**. It is the planned **first pilot** for the `deployable` platform.

### Files
| Path | What it is |
|---|---|
| `prd.md` | Product requirements: problem, metrics, 7 entities, roles, user stories, auth design, milestones |
| `features.md` | Feature list by milestone (auth, schema, CRUD, department self-serve, dashboard, integration graph, search) |
| `run_and_setup.md` | Local setup guide (below) |
| `testing-and-backup-plan.md` | **Plan only:** 40 API endpoints with zero tests, no input validation and no backup. Proposes Vitest tests, Zod validation and `pg_dump` backups |
| `app/package.json` | npm workspace root (`server` + `client`) |
| `app/docker-compose.yml` | Local PostgreSQL 16 database on port **5433** |
| `app/server/` | **Backend API**: Hono (Node + TypeScript) |
| `app/server/src/index.ts` | Starts the API and mounts all routes; `/api/health` |
| `app/server/src/middleware/auth.ts` | Login check via a JWT cookie (7 days); `requireAuth`, `requireAdmin` |
| `app/server/src/routes/` | API routes: auth, users, vendors (+ subscriptions, accounts, issues), systems, department-usages, integrations, search, dashboard |
| `app/server/prisma/schema.prisma` | **Database model:** User, Vendor, System, Subscription, Account, DepartmentUsage, SystemIntegration, OpenIssue |
| `app/server/prisma/migrations/` | Database migrations (initial 2026-04-07) |
| `app/server/prisma/seed.ts` | Example data (18 systems, usages, integrations) |
| `app/server/.env.example` | Settings to fill: database URL, Google OAuth client, JWT secret, allowed domain, ports |
| `app/client/` | **Frontend**: React 19 + Vite + React Router |
| `app/client/src/App.tsx` | Pages: Dashboard, Vendors, Systems, My Department, My Vendors, Integrations |

### Data model in plain words
| Entity | Holds |
|---|---|
| Vendor | A company Zorro buys from: status (active, prospect, churned…), Zorro owner |
| System | A tool or product from a vendor: type, status |
| Subscription | Plan, cost, billing cycle, **renewal date**, seats |
| Account | Login email/URL plus a **pointer to 1Password** (never the password) |
| DepartmentUsage | Which department uses which system, with what data, how sensitive, and whether AI is involved |
| SystemIntegration | System A → system B: method (API, Zapier, MCP…) and direction |
| OpenIssue | Open problems and TODOs per vendor |

**Roles:**
- **Admin:** can edit everything.
- **Department user:** reads everything and edits only their own department's usage.
- **Vendor owner:** manages their own vendors' issues.

### How to run (inside `app/`)
1. Start the database: `docker compose up -d`
2. Install packages: `npm install`
3. Create a Google OAuth web client with redirect `http://localhost:3000/api/auth/callback`. Then `cp server/.env.example server/.env` and fill it in.
4. Set up the database:
   ```bash
   npm run --workspace server db:migrate
   npm run --workspace server db:seed
   ```
5. Start the API (port 3000): `npm run --workspace server dev`
6. Start the client (port 5173): `npm run --workspace client dev`, then open http://localhost:5173
7. **Make yourself admin** with `npm run --workspace server db:studio` (new users are not admins).

### Gotchas
- The testing and backup plan is **not implemented**.
- The local database uses default credentials (fine locally only).
- `run_and_setup.md` contains a committed Google OAuth client ID. It's not a secret, but note it.

---

# Part 7: `departments/production/` and `departments/knowledge_management/`

## `production/AGENTS.md`: routing hub
Most of the production department's work (initiatives, skills, domain map) **moved to ZorroEng/agent-production-hub**, with its history preserved. This folder routes you:
1. If the task names a local project here, open its `AGENTS.md`.
2. If it's any other production topic, switch to agent-production-hub.
3. If unsure, ask the owner.

The table lists only `engineering_support`, and its status there is stale. `local_setup/` exists but isn't listed.

---

## Engineering Support (`production/engineering_support/`)

### Purpose
A **three-tier flow for asking engineering for help**:
- **T0:** a support person (CS, Ops…) submits an issue via Slack.
- **T1:** an **AI triage agent** classifies, enriches and routes it. *Not built yet.*
- **T2:** the on-call engineer receives it in the right Slack channel.

The current phase locks the intake form, vocabulary and **SLO policy** so T1 can be built against a stable contract.

### Status
- Intake schema v6, the vocabulary and the SLOs are **locked**.
- The v6 Slack form is ready but **not deployed**. It is blocked on 7 manual Slack List column changes.
- The **OniCall** Chrome extension is at v0.7.32.
- `solution_concept.md` and `implementation.md` are still empty skeletons.
- People: Gadi (form and SLO decisions), Nadav (reviewer), Maya (management).

### The SLO tiers (`rules/slo/tier_definitions.md`)
| Tier | Respond / fix | When the clock runs | Gate | Routed to |
|---|---|---|---|---|
| **P0 Wide** | ≤ 30 min / ≤ 4 h | 24/7 | Manager approval | Call or huddle the on-call engineer; war room |
| **P1 Scoped** | ≤ 1 h / ≤ 12 h | Sun–Thu 09–22 Israel + Fri 15–22 Israel; no overnight paging | Peer look | `#contact-engineering` |
| **P2 Degraded** | ≤ 1 / ≤ 2 Israel business days | Business days | none | `#contact-engineering-non-urgent` |
| **Backlog** | no commitment | — | — | Product requests channel |

Only P0 wakes anyone up.

### Folder map
| Path | What it is |
|---|---|
| `AGENTS.md` | Entry point: tools (`gws`), layout, "where do I go" table, conventions |
| **`.agents/`** | Project memory |
| `roadmap.md` | **The only status record**: deliverables D-1…D-13, next actions, open questions |
| `deliverables.md` | Decisions backlog from the Aug 2026 sessions |
| `run_commands.md` | Every command for OniCall |
| `learnings_invariants_bugs.md` | What broke and the rules learned from it |
| `us_lbh_worked_cases.md` | Example P1 scenarios for the Friday US window |
| `solution_concept.md`, `implementation.md` | Empty skeletons |
| **`requirements/`** | What each stakeholder needs: `motivation`, `project_leads`, `management`, `engineering` (T0/T1/T2 flow), `submitters` |
| **`rules/`** | **Authoritative policy.** A number lives here once |
| `rules/slo/tier_definitions.md` | The commitment table, escalation, on-call zones |
| `rules/slo/tier_classifying.md` | How T1 will derive P0/P1/P2 from 6 dimensions |
| `rules/times.md` | Israel and US calendars, when clocks pause |
| `rules/data_handling.md` | 4 data classes, what each surface may hold, retention |
| `rules/language_guidelines.md` | Terminology and tone |
| `rules/invariants_and_decisions.md` | Locked principles + append-only decision history |
| **`interfaces/`** | **Contracts only, no code** |
| `interfaces/slack/intake_form.md` | Authoritative form fields and allowed values |
| `interfaces/slack/ticket_list.md` | Slack List columns: "a field with no column doesn't exist" |
| `interfaces/slack/form_build_requirements.md`, `build_checklist.md` | How an engineer rebuilds the form in Slack Workflow Builder |
| `interfaces/slack/form_version_history.md`, `exports/*.json` | v4→v5→v6 history; raw exports (reference only) |
| `interfaces/chrome_extension/` | Original extension proposal + 17-field form definition |
| **`publications/`** | Human-facing material, by audience |
| `support_teams/` | Training deck, "contact engineering" guidelines, SLO policy, what's changed, form preview |
| `engineering/tiers.html` | Canonical tier reference; feedback responses |
| `leadership/engineering_support_overview.html` | Interactive overview |
| `archive/`, `assets/` | Old v5 preview; team headshots |
| **`feedback/`** | Engineering feedback, verbatim (round 1: 34 comments) |
| **`apps/support_reporter/`** | **OniCall**: the Chrome extension (below) |

### OniCall (`apps/support_reporter/`)
A **Chrome side-panel extension** used on the Zorro app to report issues with evidence. It records:
- network traffic (HAR)
- the page structure
- the user's actions
- console errors
- screenshots

It walks the submitter through a 5-step, 17-field form with P0/P1 gates. It writes the ticket to the Slack List, posts in the right channel, and uploads the evidence.

| Path | What it is |
|---|---|
| `AGENTS.md` | Decisions, layout, distribution, OAuth, version ladder (v0.1 built, v0.2/v0.3 in progress) |
| `Makefile` | `make install` / `build` / `watch` / `test` / `pack` / `patch` / `minor` … |
| `spec/` | **Source** form schema, routing branches, list mapping, evidence contract, Slack app manifest |
| `extension/` | Extension code: service worker, `core/` (tested logic), content scripts, side panel, tests, packaging tools |
| `.env.example` | Names of the Slack settings (copy to `local/.env`, which is gitignored) |

**Use:**
1. `make install`, then load `extension/` unpacked in `chrome://extensions`.
2. To release: `make patch && make pack`, upload the zip to the restricted Drive "OniCall builds" folder, and announce it in `#onicall-mvp`.

**Rules:**
- Edit `spec/`, never the generated `extension/data/`.
- Evidence can contain PHI and session cookies. PHI evidence never goes to an AI.
- Build zips contain a Slack secret, so keep the Drive folder restricted.

---

## Local Setup (`production/local_setup/`)
- **Purpose:** an agent-assisted guide to clone, install and run Zorro's **production repos `zorro-ts` and `zorro-py`** locally.
- **Status:** active, v1 tested on 2026-06-30 (Apple Silicon Mac). Owner: Maya C.

| File | What it is |
|---|---|
| `AGENTS.md` | Sources, guardrails (agents never read `.env` files) |
| `execution/local-dev-setup.md` | **The step-by-step guide** |

**Steps in the guide:**
1. Homebrew and Xcode tools.
2. `gh repo clone` both repos.
3. Node via nvm, plus pnpm and nx.
4. Python 3.13 via uv.
5. AWS CLI, psql and `akl`.
6. Docker.
7. `.env` files: they arrive as an **encrypted zip from Nadav**, with the password sent separately.
8. Start Postgres with docker compose.
9. Reset the database with Prisma and nx. **This destroys local data; local only.**
10. `aws login` and `akl`.
11. Start the servers:

| Service | Command | Port |
|---|---|---|
| FastAPI Agent Hub (zorro-py) | `nx run pymonolith:serve` | 8000 |
| Decision Backoffice (zorro-py) | `nx run decision-backoffice:serve` | 3000 |
| Monolith API (zorro-ts) | `pnpm monolith:dev` | 3333 |
| App (zorro-ts) | `pnpm app:dev` | 3009 |

*Useful to you: the `zorro-ts` repo you have open is one of these production repos.*

---

## `departments/knowledge_management/`
Only a **"Moved" stub** (`README.md`). The department and its `kb_templates` now live in **ZorroEng/agent-production-hub**.

---

# Part 8: `departments/cross/`, cross-department projects

The only folder here is **`zorro_deck/`**.

There is **no moved stub for `zorro_html_deck_template`** here, even though the root `AGENTS.md` mentions one. That skill lives in ZorroEng/agent-production-hub.

## Zorro Deck (`departments/cross/zorro_deck/`)

### Purpose
A local agent harness plus a design system for Zorro presentations. **"The deck is data":**
- Each slide is written once as a structured JSON spec (content + layout + graphical elements).
- The spec renders to **HTML** (fast agent iteration), **PPTX** (offline and client hand-off) and **Google Slides** (the collaboration copy).
- The three outputs are renders of one spec, never separate sources.

### Status
- `VERSION` = **0.3.4**, which is authoritative.
- `package.json` and `tokens.json` say 0.3.0, and the `AGENTS.md` status line says v0.2.0. Both are stale.
- No owner is named.

### Roadmap (`.agents/roadmap.md`)
| Phase | Goal | Status |
|---|---|---|
| 0 | Scaffold | done (v0.1.0) |
| 1 | Pull and catalogue the 84-slide `zorro_gallery` | done |
| 2 | HTML version of the gallery | done (84/84) |
| 3 | Promote to the design system | partly done (tokens, engine, roadmap block) |
| 4 | Recipes in `templates/` | not started |
| 5 | `projects/investor_deck` | **done (v0.3.4)**: 45 slides 1:1, native shapes, animations |
| 6 | Round-trip (re-pull from Drive and diff) | not started |
| 7 | Add `zorro_html_deck_template` as a second reference | later |
| 8 | Make it deployable | later |

### Root files
| File | What it is |
|---|---|
| `AGENTS.md` | Guide and router: gws rules, slide model, layout, 8-step workflow, guardrails |
| `VERSION` | `0.3.4`, stamped into renders and published file names |
| `package.json` | npm package `zorro-deck`. Scripts: `typecheck`, `gallery`, `build`, `publish`, `shoot`, `review`. Uses pptxgenjs, playwright-core, sharp, fontkit, jszip |
| `tsconfig.json` | Strict TypeScript config |
| `.gitignore` | Ignores `dist/`, pulled sources, and nearly all of `projects/investor_deck` (confidential) |

### `.agents/`: planning and knowledge
| File | What it is |
|---|---|
| `roadmap.md` | The phase table above and open questions |
| `taxonomy_notes.md` | Lessons from each reference: grid (12 columns, 59.5 pt margins), brand ramp teal `#12B5A8` → violet `#8033EF`, gradients faked as ~20 solid segments, the Slides API has no animation data, image sourcing order |
| `ideation/ref_proposals/` | What Zorro Deck should learn from `sales/proposals`: architecture, brand comparison (font conflict: TWK Lausanne vs Inter) |

### `design_system/`: the reusable look and parts
| Path | What it is |
|---|---|
| `schema/deck.schema.json` | JSON Schema for a deck: `meta` (title, recipe, Drive IDs, owner…) plus `slides[]` (id, archetype, content, theme, variant, layout, elements, speaker notes) |
| `schema/catalogue.schema.json` | Shape of a reference catalogue or component registry |
| `schema/README.md` | How a slide resolves (archetype → style → variant → theme → mappers). Rules: content holds no style; units are points |
| `tokens/tokens.json` | Design tokens: page 960×540, colours, brand gradient, type (Inter), spacing, radius, shadows, themes light/tint/lavender/dark/glow |
| `components/*.ts`, `html.css` | HTML component path: core types, reusable elements (chips, cards, bullets…), the 9 gallery archetypes (cover, agenda, intro, statement, quote, divider, cards, pillars, side_by_side), registry |
| `engine/` | **The layout engine.** `layout.ts` (frames, grid, themes), `measure.ts` (text measuring), and serializers `html.ts`, `pptx.ts` (native editable PPTX + animations) and `gslides.ts` (Slides API requests). `images.ts` crops and masks images; `blocks/roadmap.ts` is the first promoted block |
| `templates/README.md` | Planned "recipes" (general, product, tech, investor, brief). None exist yet |

### `harness/`: build, publish, review tools
| Path | What it is |
|---|---|
| `src/cli.ts`, `src/deck.ts` | `build <project>`: loads `spec.json` and the project layouts, checks the slide outline, writes `dist/deck.html`, `dist/deck.pptx`, `layout.json`, `outline.md` |
| `src/publish.ts` | Builds, uploads a seed PPTX converted to Google Slides, adds the remaining shapes and text natively through `batchUpdate`, and uploads the `.pptx` and `.html` companions. Re-publishing keeps the same links |
| `src/review.ts` | Visual review: original vs rebuild thumbnails and HTML screenshots, plus a lint for overflow, margins and overlaps → `dist/review/index.html` |
| `src/shoot.ts`, `src/browser.ts` | Screenshot slides with headless Chrome |
| `tools/pull_deck.py` | Pull a Google Slides deck (read-only) via gws: JSON, PPTX export, thumbnails |
| `tools/inventory.py`, `inventory_pptx.py` | Turn a deck into `inventory.json` (layouts, fonts, colours) |
| `tools/read_animations.py` | Read click animations from PPTX exports |
| `exporters/` | Empty placeholder. The real serializers are in `design_system/engine/` |

### `projects/investor_deck/` (confidential)
Only `AGENTS.md`, `README.md` and `components/` are in git. The spec, assets and outputs stay local and in the InvestorDeck shared drive.
- `components/slides.ts`: about 23 project layouts (kpi_grid, chart, matrix, roadmap, team_grid, closing…) waiting to be promoted into the design system.
- `components/flow.ts`: node + connector diagrams.

### `references/zorro_gallery/`: the 84-slide reference template
| Path | What it is |
|---|---|
| `README.md` | Where the source PPTX lives, how to re-pull it, and the slide families (Cover A–F, Agenda A–C, Cards A–E…) |
| `catalogue.json`, `inventory.json` | Catalogue of archetypes and styles; machine inventory |
| `source/thumbs/` | 84 thumbnails |
| `html/` | `gallery.spec.json` (84 slides as spec), `render.ts` (`npm run gallery`, also the regression check), `assets/` (logos, icons, backgrounds) |

### How it works
```
spec.json → laySlide() → layout nodes (pt) ─┬→ deck.html
                                            ├→ deck.pptx
                                            └→ Google Slides (seed PPTX + batchUpdate)
```
1. Pull a reference.
2. Catalogue it.
3. Promote components into the design system.
4. Write a recipe.
5. Build the project spec.
6. Build, review and publish.
7. Round-trip edits made in Drive back into the spec (not built yet).

### Key commands (inside `zorro_deck/`)
```bash
npm run typecheck
npm run gallery
npm run build   -- projects/investor_deck
npm run publish -- projects/investor_deck
npm run review  -- projects/investor_deck [--only 6,9]
python3 harness/tools/pull_deck.py <presentationId> <dir>
```

### Rules
- **All Google work goes through `gws`.** No browser automation or other SDKs.
- **Drive writes go outward.** Confirm before the first write and before publishing an investor version.
- **Never fork content.** The spec is the only source; Drive edits get pulled back.
- **Content holds data, never style.** Variants are chosen by rules.
- **Draw in code from tokens, not pasted images.** Screenshots stay images.
- **Units are points** on a 960×540 page.
- **Bump `VERSION`** on every pass.
- **Investor deck content is never committed.**

### Known inconsistencies
- Version numbers disagree across files.
- The `AGENTS.md` layout tree is out of date.
- `core.ts` references a missing `scene.ts`.
- `templates/` and the component subfolders are empty placeholders.

---

# Part 9: `deployable/`, the planned hosting platform (root-level project)

## What it is
A plan for **one monorepo that builds and deploys every agentlab server app** to infrastructure that mirrors Zorro production.
- Each app is either **internal** (Zorro staff, Google Workspace sign-in) or **external** (brokers, clients and prospects, signing in with Frontegg and/or per-link tokens).
- Each app runs in a **test** and a **prod** environment.

**Status: design only** (draft 2026-09-28). **No app code has moved yet.** Decision D4 says nothing moves until the design is reviewed.

## Files

| File | What it is |
|---|---|
| `AGENTS.md` | Scope: work only inside `deployable/`. Department modules are read-only from here. Moving an app in needs that module owner's agreement. |
| `.agents/design.md` | The full design (source of truth), summarized below |
| `.agents/decisions.md` | Decision log, D1–D9 |

## Key points of `design.md`

### 1. What can be deployed
Most agentlab projects are **not servers**: they are Chrome extensions, a desktop app, CLIs and static files. The candidates are:

| Candidate | From | Exposure | PHI | Readiness |
|---|---|---|---|---|
| Extension server (new) | `deployable/apps/extension-server` | internal | no | designed here |
| Systems Manager | `general_admin/systems_manager/app` | internal | no | working app, local only |
| Carrier registry API | `operations/carrier_enrollment/harness/carrier_registry/server` | internal | no | handler stubs |
| OniCall evidence service | `production/engineering_support` roadmap | internal | **yes** | not started |
| Hosted Census Raccoon | `sales/census_ingestion/census_ui` | internal | **yes** | desktop app today |
| Proposals studio + hosting | `sales/proposals` | internal + **external** | low | built as a proof of concept |

Out of scope: the Census Raccoon DMG, the carrier CLI and the ai_dept scripts. These keep their current distribution.

### 2. Target platform: copy Zorro production
- Nx + pnpm monorepo, one Dockerfile per app
- AWS EKS with Helm, AWS ALB ingress
- Secrets Manager → ExternalSecrets, Cloudflare DNS, Pulumi
- Preferred placement: an `agentlab` namespace on Zorro's TEST and PROD clusters (open question Q1)

### 3. Planned layout
```
deployable/
  apps/<app>/          app.yaml (name, exposure, auth, phi, department) + Dockerfile + code
  packages/
    auth-internal/     verify Google SSO identity from the load balancer
    auth-external/     Frontegg + per-link token verification
    platform-config/   env loading, health checks, PHI-redacting logger
  deploy/
    chart/             one generic Helm chart
    environments/{test,prod}/
    infra/             Pulumi
```
Each app's `app.yaml` declares its exposure, and CI enforces it. For example, an app marked `phi: true` cannot be external.

### 4. Auth model
| | Internal apps | External apps |
|---|---|---|
| Who | Zorro staff | brokers, clients, prospects |
| How | Google SSO at the AWS load balancer, re-verified in the app | **Frontegg** (named, returning users) and/or **sealed per-link tokens** (one-off recipients) |
| Hostnames | `<app>.agentlab.myzorro.co` | product-facing names, with untrusted content on a separate domain |

Machine clients (Chrome extensions, CLIs) send a Google ID token.

### 5. Extension server
One internal service storing every Zorro Chrome extension build and serving auto-updates for all of them.
- Downloads are signed (Ed25519, with the key in AWS KMS) and checked with sha256.
- There's a rollback path and beta/stable channels.
- The update helper on the operator's Mac reuses the Go updater from carrier_enrollment.

### 6. Proposals
Two apps:
- **`proposals-studio`** (internal): AEs build proposals.
- **`proposals-hosting`** (external): prospects open links.

Publishing writes immutable files to S3 and creates a link token that can be revoked. Employee data from Tornado dumps never reaches the hosting app.

### 7. Environments
- **test:** deploys on merge to `main`, with **synthetic data only**.
- **prod:** gets the same image, promoted by hand.

### 8. Compliance
- No static secrets.
- PHI goes to LLMs only through AWS Bedrock in Zorro's account.
- PHI apps get encrypted stores, redacted logs and an audit trail.
- The BAA scope question must be settled before any PHI app reaches prod.

### 9. Migration plan
| Phase | Work |
|---|---|
| 0 | Platform skeleton: two "hello" apps (internal and external) live in test |
| 1 | Pilot: Systems Manager |
| 2 | Extension server |
| 3 | Carrier registry API |
| 4 | OniCall evidence service |
| 5 | Hosted Census Raccoon (needs BAA sign-off) |
| 6 | Proposals studio and hosting |

**Open questions Q1–Q10** cover: cluster placement, ALB vs Cloudflare Access, the Frontegg tenant, no-PHI-in-test, hostnames, extension update paths, and Tornado dump retention.

## Decision log (D1–D9)

| # | Decision |
|---|---|
| D1 | Hosting mirrors Zorro production, plus a separate test environment |
| D2 | Internal apps use Google Workspace SSO |
| D3 | External apps use Frontegg and/or per-link tokens |
| D4 | The design doc comes first; no code moves until it is reviewed |
| D5 | The project lives at the repo root as `deployable/` |
| D6 | Proposal hosting is built here, not in Tornado |
| D7 | Hosted Census Raccoon uses Google SSO (replacing an earlier Cognito plan) |
| D8 | One internal extension server for all Zorro Chrome extensions |
| D9 | Proposals has a studio app and a hosting app, with a publish flow between them |

---

# Part 10 (appendix): docs that are out of date

The repo moves fast, so several docs lag behind the code. **When a doc and the code disagree, trust the code and `VERSION` files, and ask the owner.**

| Area | Out-of-date doc |
|---|---|
| Root | `AGENTS.md` mentions a `cross/skills/zorro_html_deck_template/` stub that doesn't exist. `marketing` is listed but has no folder |
| ai_projects_setup | `service_account.sh` mentions an `install.sh` that isn't there. The template `connector.conf` comment says `key \| impersonate`, but the wrapper accepts `impersonate \| keychain-key` |
| ai_workstation_setup | `verify-gws-auth.sh` points to an old doc path. `lst`'s DESIGN.md refers to a missing README |
| census_ingestion | `census_ui/README.md` describes the early skeleton. The workspace path in `distributions.md` is old. Template and profiler paths are slightly off in the harness docs |
| proposals | `AGENTS.md` status says "Skeleton (2026-07-02)". Several docs use v0.4/v0.6/v0.7 names; the current version is v0.8.31 |
| carrier_enrollment | Harness docs mention `bcbs_ri` and `carrier_profiles/`, which no longer exist. `make help` doesn't list all targets |
| unified_crm_cockpit | `VERSION` 0.14.0, but `AGENTS.md` says 0.13.0 and `package.json` says 0.12.0. Some docs still name the old `.py` scripts |
| compliance | Docs mention `--log` / `--replace` flags that don't exist. Arial vs Roboto font. Status text is stale |
| production | `engineering_support` status in the hub is stale. `local_setup` isn't listed. Test counts disagree (92 vs 213) |
| zorro_deck | `VERSION` 0.3.4 vs 0.3.0 / v0.2.0 elsewhere. The `AGENTS.md` layout tree is outdated |
