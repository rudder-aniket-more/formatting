# agentlab: complete guide

*Everything about every project in `ZorroEng/agentlab`: what it is, who uses it, how it's used, inputs and outputs, whether it's live or file-based, credentials, worked examples, and step-by-step procedures by role. Based on the repo's own docs, Oct 2026.*

**In five lines:**
- agentlab is Zorro's repository for **AI agent projects across departments**.
- Each project solves one business problem. Claude does the work by following the project's written rules (`AGENTS.md`) and scripts; people review and approve.
- Projects are a mix of **apps** (Mac app, Chrome extensions, web app, offline HTML tools), **agent projects** (you ask Claude), **generated reports** and **process docs**.
- Patient and personal data stays out of the repo; agents have their own limited Google identities and never contact outsiders.
- Most apps run locally or from Google Drive today; a hosting platform ("Deployable") is designed but not built.

**Related files:** `agentlab-summary.html` (one-page summary to share) · `agentlab-talking-points.md` (speaking notes) · `agentlab-reference.md` (file-by-file technical map).

> **About the examples.** They show realistic tasks based on each project's docs. All names, companies and numbers in them are made up.

## Contents
- **Part 1: Basics.** How every project is used, map of departments, overview table of all projects
- **Part 2: Projects.** One section per project, by department
- **Part 3: Where credentials live**
- **Part 4: Procedures by role.** Learner, end user, operator, builder, contributor, reviewer, AI Dept, publishing
- **Quick reference: which project for which need**

---

# Part 1: Basics

## How every project is used

Almost every project works the same way:

```
1. Open Claude Code in agentlab (or in your ~/Workspaces for operator projects)
2. "Read the root AGENTS.md. Department: <dept>, project: <project>."
3. Claude reads that project's AGENTS.md (its rules and workflows)
4. You give a task in plain English
5. Claude runs the project's scripts, asks you at decision points, and saves results
   (code → GitHub PR, documents → Google Drive, personal data → stays on your machine)
```

The project-specific parts are **what the project does, who it's for, and its rules**. That is what this guide covers.

## Map

| Department | Projects |
|---|---|
| **AI Dept** | AI Initiative Shop · AI Org Deck · AI Projects Setup · AI Workstation Setup · Investor Materials |
| **Sales** | Census Ingestion (Census Raccoon) · Proposals (Proposal Studio) |
| **Operations** | Carrier Enrollment (Enrollment Copilot) · Unified CRM Cockpit ("Zong") · Customer Teams (4 projects) |
| **Compliance** | AI Policy |
| **General Admin** | Systems Manager |
| **Production** | Engineering Support (incl. OniCall) · Local Setup |
| **Cross** | Zorro Deck |
| **Knowledge Management** | *moved to another repo* |
| **Marketing** | *reserved, no projects yet* |
| **Root project** | Deployable |


## Overview of all projects

*The § column is the section number in Part 2.*

**Kinds of project:**

| Kind | Meaning |
|---|---|
| 🖥️ **App** | Software people open and click through: Mac app, web app, Chrome extension, offline HTML tool |
| 🤖 **Agent project** | No app. A person asks Claude in plain English, and Claude follows the project's written rules and scripts |
| 📊 **Generated report** | A script or pipeline builds a file (HTML dashboard, deck) that people then open |
| 📄 **Process / docs** | Written rules, guides or plans. Nothing runs |

**Data connection:**
- 🔴 **Live**: reads or writes another system while it runs.
- 🟡 **On-demand pull**: fetches fresh data when someone runs it, then works from that copy.
- 🟢 **File-based**: works only from files or text someone gives it.

| § | Project · Dept · Kind | What it does · Who uses it | How it's used (input → steps → output) | Data connection | Credentials |
|---|---|---|---|---|---|
| 1.1 | **AI Initiative Shop**<br>AI Dept<br>📄 Process / docs | The approval route for new AI ideas: decides owner, support level, data risk, fallback and how to build.<br>**Users:** any employee submits; dept leads + AI Dept evaluate | **Input:** problem + example + quick evaluation, by any employee, in the "AI Initiative Candidates" Google Doc.<br>**Steps:** submit → scored on 5 points → PHI goes to Compliance first → time-boxed proof of concept → release (needs runbook + support contact) → maintain.<br>**Output:** a go/no-go decision; approved ideas become a project | 🟢 Manual (Google Doc) | Zorro Google account |
| 1.2 | **AI Org Deck**<br>AI Dept<br>📊 Generated report (slides) | Exec decks explaining Zorro's AI approach.<br>**Users:** AI Dept presents to leadership and teams | **Input:** slide text in `deck.md` (AI Dept) + initiatives from the Candidates Doc.<br>**Steps:** edit text → `python build_deck.py`.<br>**Output:** **PowerPoint + PDF** | 🟡 Pulls the Doc on rebuild | Your Google login (`gws`) |
| 1.3 | **AI Projects Setup**<br>AI Dept<br>📄 Process + setup scripts | Gives each agent project **its own Google robot account** with access to specific folders only; defines the standard project folder layout.<br>**Users:** AI Dept (creates), builders (request), agents (use) | **Input:** project name, needed Drive folders, who may use it, from the builder.<br>**Steps:** AI Dept creates the account → shares the folders (Viewer by default) → grants the person → person runs `setup-google.sh <project>`.<br>**Output:** a working, verified **Google identity for that project**; `gws` inside the project acts as it | 🔴 Live (each Google call gets a fresh 15-min token) | GCP admin (AI Dept), 1Password, your `oauth2l` sign-in |
| 1.4 | **AI Workstation Setup**<br>AI Dept<br>🖥️ Setup script + CLI tools | Sets up a **person's Mac** with the right AI tools for their role and proves the access is correct. Includes `lst`, the task scheduler.<br>**Users:** new staff, AI Dept | **Input:** the person's role/tier + their logins and approvals.<br>**Steps:** request access → run `setup-workspace.sh` (or `install.sh --profile <tier>`) → do the manual logins → run the verify script → optionally `lst install <task>`.<br>**Output:** a **configured Mac** with `~/Workspaces`, a verification report, scheduled tasks | 🔴 Live (installs + login checks) | Claude seat, Google group, 1Password, GCP project access, Slack, (GitHub PAT for builders/engineers) |
| 1.5 | **Investor Materials**<br>AI Dept<br>📄 Docs | The official AI story for investors and sales.<br>**Users:** leadership, marketing | **Input:** AI achievements + the investor PPTX, from leadership/AI Dept.<br>**Steps:** read the pillars and slide map → write AI content into the deck.<br>**Output:** updated investor **deck / text** | 🟢 File-based | None |
| 2.1 | **Census Raccoon**<br>Sales<br>🖥️ **Mac app** (AI inside) | Converts any broker's employee census spreadsheet into **Zorro's standard census format** for quoting.<br>**Users:** **AEs**; engineers maintain it | **Input:** broker census files (xlsx/csv/pdf), dropped in by the **AE**.<br>**Steps:** install `.dmg` from Drive → paste Bedrock key once → New enrollment → drop files → Profile (AI) → **AE reviews** → Extract (AI) → Normalize (code) → Validate (code) → Output (AI) → **AE reviews** → answer decision questions along the way.<br>**Output:** **Excel file `normalized.xlsx`** (Employees + Plans) + summary → AE uploads it to Tornado | 🟢 File-based; AI calls go live to AWS Bedrock; **no live Tornado link** | **AWS Bedrock key** (1Password → Keychain) |
| 2.2 | **Proposal Studio**<br>Sales<br>🖥️ **Offline HTML app** (one file, no install) | Builds **branded sales proposals** from a quote. It replaced Qwilr.<br>**Users:** **AEs** build them; **prospects** read them; builders maintain the template | **Input:** Tornado quote export (JSON) + AE's choices, by the **AE**.<br>**Steps:** download Studio HTML from Drive → open in Chrome → pick your AE identity → load the quote → choose plans, confirm fees, edit text, toggle slides → download.<br>**Output:** **interactive HTML proposal + PDF** → AE emails them to the prospect | 🟢 File-based, fully offline; **no live Tornado link** | AEs: Drive + Tornado access. No keys inside the app |
| 3.1 | **Enrollment Copilot**<br>Operations<br>🖥️ **Chrome extension** | **Pre-fills insurance carriers' enrollment forms** from Zorro data, so operators review instead of retyping sensitive data.<br>**Users:** **Ops operators**; engineers write per-carrier "recipes" | **Input:** the member's enrollment record (loaded by the **operator**) + the carrier's recipe (by **engineers**).<br>**Steps:** open the dedicated Chrome profile → log in to the carrier site yourself → open the side panel → load member → on each page click **Fill**, check, Next → **operator submits**.<br>**Output:** **completed enrollment on the carrier's website** + session recording saved to Drive | 🔴 Live on the carrier website. Live production data pull **not yet cleared** for real use | Carrier logins (Ops 1Password, typed by hand), 2FA via Slack, Google robot key bundled in the extension |
| 3.2 | **CRM Cockpit ("Zong")**<br>Operations<br>📊 **Generated report** (offline HTML dashboard) | **One dashboard of customer issues** from Intercom: open items, response times, problem companies/brokers/carriers, AI summaries, CSM quality checks.<br>**Users:** **Ops managers** view; engineers/AI Dept build | **Input:** Intercom conversations, tickets, calls; the pull is run by an **engineer / AI Dept**.<br>**Steps:** `npm run pack` (pull → analyze → enrich → adapt → optional AI summaries → build) → `publish` to Drive → managers download and open in a browser.<br>**Output:** **one HTML dashboard file** (+ PDF reports, Gap Report Doc tab) | 🟡 On-demand pull. The dashboard is a **dated snapshot, not live** | Intercom API token (`.env`), Google login to publish |
| 3.3a | **Enrollment Dashboard**<br>Operations (team)<br>🤖 Agent project → 📊 HTML dashboard | Builds the **leadership enrollment dashboard**: employees, lives, carriers, states, money.<br>**Users:** **Bradley** runs it; **leadership** views it | **Input:** 2 Metabase exports, downloaded by **Bradley**.<br>**Steps:** tell Claude "new data, build next version" → scripts compute every number → Claude builds the HTML and compares with the last version → **Bradley approves** → Claude overwrites the live Drive file.<br>**Output:** **HTML dashboard at one permanent Drive link** | 🟢 File-based (manual downloads); publishing to Drive is live | Metabase, project Google account, file ID in 1Password |
| 3.3b | **Help Center Sync**<br>Operations (team)<br>🤖 Agent project → Google Doc + Sheet | Finds **help-center articles and SOPs that a change has made outdated**, and proposes exact edits.<br>**Users:** **Lerone** (Knowledge Center owner) | **Input:** change description, type, effective date (+ PRD/Loom/Slack links), from **Lerone** or the change owner.<br>**Steps:** tell Claude the change → it reads all KC docs, SOPs and the live help site → writes findings (Remove / Rewrite / Screenshots only / New needed / Flag…) → a person copies approved text into Intercom.<br>**Output:** **proposal Google Doc + tracker Google Sheet** | 🔴 Live reads of Google docs and the help site | Project Google account |
| 3.3c | **ICHRA Takeover Objections**<br>Operations (team)<br>🤖 Agent project → deck + Doc | Prepares **accurate answers for prospects switching from a competitor** to Zorro.<br>**Users:** **Ann Marie** (owner), **Bradley** (Ops reviewer), the deal's **AE** | **Input:** the prospect's questions, the competitor's claims, census counts, per deal, from the **AE / Ann Marie**.<br>**Steps:** Claude matches claims against the approved answer bank and fact rules → drafts → unsure items marked `[NEEDS CONFIRM]` → **Ops + owner approve** → **AE sends**.<br>**Output:** **transition deck (PDF), Q&A Google Doc, or reply text** | 🟢 File-based, one folder per deal | Project Google account |
| 3.3d | **CS Recruiting**<br>Operations (team)<br>🤖 Agent project, **scheduled** → Sheets + Slack | **Screens job applicants** for CS/Implementation roles and keeps the interview tracker current.<br>**Users:** **Ann Marie, Bradley** | **Input:** applications arrive in the careers sheet from the website form (by **candidates**, automatic); interviews come from Calendar.<br>**Steps:** runs automatically 3×/day → reads new applicants → scores fit (US-based required) → adds rows → updates the tracker → DMs top candidates to the team. **People make every hiring decision.**<br>**Output:** **rows in Google Sheets + Slack DM** | 🔴 Live, on a schedule | Project Google account, HubSpot token (in progress), Slack |
| 4.1 | **AI Policy**<br>Compliance<br>📄 Docs + Python publishing tool | Writes and publishes Zorro's **AI and data policies**, with an approval record.<br>**Users:** compliance team and policy owners write and approve; all staff read | **Input:** policy text (Markdown) by **compliance / owners**; register details in the sheet; reviewers' comments.<br>**Steps:** write draft → `push_to_drive.py --push` (updates the one draft Doc, saves comments) → reviewers comment → owner approves in the sheet → `--push --publish`.<br>**Output:** **Google Docs** (living draft + read-only published copy) + **Decision Log rows in a Google Sheet** | 🔴 Live sync with Drive/Sheets when run | Your own Google login + edit access to the Policies folders |
| 5.1 | **Systems Manager**<br>General Admin<br>🖥️ **Web app** (database, **no AI**) | Replaces spreadsheets: **who owns each software tool, its cost, renewal, which departments use it, what data it touches, whether it uses AI**.<br>**Users:** admins, department users, vendor owners | **Input:** vendors, systems, costs, usage, **typed into forms** by admins and department users.<br>**Steps:** sign in with Zorro Google → pick department → browse/search → record "my department uses X with Y data" → check the dashboard.<br>**Output:** a **web app** to search and filter (runs locally today) | 🔴 Live database, **manual entry** (no auto-sync) | Google OAuth client, JWT secret, database URL (`server/.env`) |
| 6.1 | **Engineering Support + OniCall**<br>Production<br>📄 Process + 🖥️ **Chrome extension** + Slack form | How support staff **report production problems to engineering**, with priorities P0–P2 and response times. **OniCall** records the bug as evidence.<br>**Users:** CS/Ops/support report; on-call engineers respond | **Input:** a bug report + evidence captured from the browser, by **support staff**.<br>**Steps:** open OniCall → Record → reproduce → Stop → fill the 5-step form, pick priority (P0 needs a manager) → Submit.<br>**Output:** **ticket in a Slack List + post in the right Slack channel + evidence files** | 🔴 Live (posts to Slack) | Slack app credentials (`local/.env`), users' own Slack |
| 6.2 | **Local Setup**<br>Production<br>📄 Guide (Claude walks you through) | Gets **Zorro's production code (zorro-ts, zorro-py)** running on your own machine.<br>**Users:** engineers | **Input:** GitHub repos, `.env` zip, AWS login, from the **engineer** (+ Nadav sends the env zip).<br>**Steps:** ask Claude for local_setup → install tools → clone → decrypt `.env` → start DB → reset → `aws login` → start the 4 services.<br>**Output:** **app and APIs running locally** (ports 8000, 3000, 3333, 3009) | 🔴 Live (GitHub, AWS) | GitHub, AWS, encrypted `.env` zip |
| 7.1 | **Zorro Deck**<br>Cross<br>🖥️ **CLI tool** (TypeScript) → 📊 slides | Builds **Zorro presentations from one source file** so the HTML, PowerPoint and Google Slides versions always match.<br>**Users:** deck builders (first: investor deck) | **Input:** slide content in `spec.json` (+ images, existing deck) by the **deck builder**.<br>**Steps:** pull the existing deck → edit `spec.json` → `npm run build` → `npm run review` → `npm run publish`.<br>**Output:** **HTML + PowerPoint + Google Slides** (same link on re-publish) | 🟢 File-based build; publish to Slides on demand | Your Google login + Drive folder access |
| 9 | **Deployable**<br>Root<br>📄 Design doc | **Plan** to host agentlab apps on Zorro's AWS (test + prod), with Google sign-in for staff and secure links for clients.<br>**Users:** AI Dept, platform team | **Input:** decisions from platform, compliance and product owners.<br>**Steps:** answer 10 open questions → build platform → move apps one by one (Systems Manager first).<br>**Output:** today **design docs only**; later **hosted web apps** | — | None yet |

---

# Part 2: Projects

# 1. AI Dept (`departments/ai_dept/`)

**What the department does:** the AI department decides **which AI projects Zorro builds**, **sets up people and agents** safely, and **explains AI** to leadership and investors. Its projects are mostly about the *system* the other departments' projects run on.

---

## 1.1 AI Initiative Shop (`ai_initiative_shop/`)

**What it is:** the process for turning a department's problem into a supported AI tool. It's a set of rules and documents, not software.

**Who uses it:** any employee with an idea; department leads and the AI department, who evaluate ideas.

**Status:** active process.

**How it's used:**
1. **Submit:** add your problem to your department's tab in the Google Doc "AI Initiative Candidates". Include:
   - what the pain is
   - an example case
   - a quick evaluation: size, severity, how often it happens, timing
2. **Evaluate:** the AI department and your department lead score it on 5 points:

   | Question | Options |
   |---|---|
   | Who owns it? | department / co-owned / AI Dept / vendor |
   | How much support does it need? | T0 experimental → T3 mission-critical |
   | How sensitive is the data? | public → internal → confidential → **patient data (PHI)** |
   | What happens if it breaks? | do it manually / cached / failover / hard stop |
   | Which way do we build it? | **A** Cassidy (no-code SaaS) · **B** Claude hands-on · **C** custom app · **D** Google Apps Script / Gem |

3. **Proof of concept:** time-boxed to 2–6 weeks.
4. **Release:** needs a runbook and a support contact.
5. **Maintain:** experimental tools are reviewed quarterly, then retired or promoted.

**Rule:** if any of the 5 questions has no answer, the idea doesn't move forward. PHI ideas go to Compliance **before** any proof of concept.

**Example:**
> Ops says: "We spend 3 hours a day re-typing member data into carrier websites."
> - Submitted with an example and its frequency (20 enrollments a day).
> - Scored as: AI-Dept-owned · T2 · **PHI** · hard-stop fallback · path C (custom app).
> - Because it involves PHI, Compliance reviews it first.
> - It becomes the **Carrier Enrollment** project (section 3.1). This is one of the worked examples in `lifecycle.md`.

**Key files:**
- `lifecycle.md`: the 5 phases
- `candidate-evaluation-criteria.md`: the scoring
- `systems-paths.md`: the 4 build paths
- `risks.md`: the risks the process manages
- `initiatives/phi-ready-llm-access.md`: how sales can use AI on PHI safely through AWS Bedrock

#### At a glance: kind, data connection and credentials

**Kind:** 📄 Process / docs · **Data connection:** 🟢 Manual (Google Doc)

**Credentials:**
  - Your Zorro Google account, with access to the Candidates Doc.
  - To rebuild the deck: Python with python-pptx. No secrets.

---

## 1.2 AI Org Deck (`ai_org_deck/`)

**What it is:** two executive slide decks.
- **"Building an AI-Ready Organization"** (10 slides): agent loops, engineering examples, how AI is managed across Zorro.
- **"AI Across Zorro"** (7 slides): a simpler version for non-engineers. It introduces 5 "tracks": Workflow Automation, Knowledge & Search, PHI / Systems-of-Record, Insight & Analytics, Creative & Content.

**Who uses it:** the AI department, presenting to leadership and other departments.

**How it's used:**
1. Edit the slide text in `deck.md`.
2. Rebuild the PowerPoint and PDF with the Python script.

**Example:**
> You → Claude: *"Update slide 3 of the overview deck with the latest initiatives from the AI Initiative Candidates doc, then rebuild it."*
>
> Claude pulls the Google Doc with `gws`, edits `overview/deck.md`, and runs `python build_deck.py`. You get a new `ai-org-overview-exec.pptx` and `.pdf`.

**Command:** `/tmp/aisaip-venv/bin/python build_deck.py` (needs python-pptx).

#### At a glance: kind, data connection and credentials

**Kind:** 📊 Generated report (slides) · **Data connection:** 🟡 Pulls the Doc on rebuild

**Credentials:** `gws` with your Google login (to read the Doc). Python with python-pptx.

---

## 1.3 AI Projects Setup (`ai_projects_setup/`)

**What it is:** how **agents** (not people) get access to systems. It has two parts:
- **Google connector:** every agent project gets its **own Google robot account** (a "service account"), with access only to specific Drive folders. Agents never use a person's login.
- **Project standard:** the folder template every agent project follows.

**Who uses it:** the AI department (creates the accounts), builders (create projects), and agents (read the rules).

**How it's used:**
1. A builder asks the AI department for a Google account for their project.
2. The AI department:
   - creates `<project>-agent@internal-ai-systems...`
   - shares only the needed Drive folders with it, as Viewer by default
   - grants the builder permission to use it
3. The project's `connectors/google/connector.conf` names that account.
4. On the user's Mac, `setup-google.sh <project>` signs in once (keyless: it creates 15-minute tokens).
5. From then on, when Claude runs `gws` **inside that project folder**, it acts as the project's account. The `gws` wrapper **blocks** Gmail, sharing changes and calendar writes.

**Example:**
> The help-center-sync project needs to read the Knowledge Center docs.
> - The AI department creates `help-center-sync-agent` and gives it Viewer access to the KC folder.
> - Lerone runs `setup-google.sh help-center-sync` once.
> - Claude then reads the docs with `gws docs documents get ...`, and every call prints `gws: as help-center-sync-agent`.
> - If Claude tried `gws gmail ...`, the wrapper would refuse.

**The project standard (what every project folder looks like):**
```
<project>/
  AGENTS.md        instructions for the agent (team, rules, who never to contact)
  HANDOFF.md       plain-English guide for the human owner
  .env.example     list of secrets needed (as 1Password references, no values)
  harness/         rules/ · skills/ · tools/ · schedules/ · state/
  connectors/      google/ (identity + access map), slack/, ...
  .agents/         mission, learnings, roadmap, run_log
  assets/          one folder per run: inputs/ (never changed) + outputs/
```

#### At a glance: kind, data connection and credentials

**Kind:** 📄 Process + setup scripts · **Data connection:** 🔴 Live (each Google call gets a fresh 15-min token)

**Credentials:**

| Credential | Who needs it | Where it lives |
|---|---|---|
| GCP admin access to the `internal-ai-systems` project (`gcloud`) | AI Dept admin, to create accounts | Admin's own login |
| 1Password CLI, **AI Vault / AI Builders** vault | Admin (stores keys); users (keyless sign-in app) | 1Password |
| **Keyless (preferred):** your `oauth2l` sign-in + the "Token Creator" role on that account | Each person running the project | `~/.config/oauth2l/` (your own sign-in) |
| **Fallback:** the service-account key | Each person | macOS Keychain (from 1Password) |

---

## 1.4 AI Workstation Setup (`ai_workstation_setup/`)

**What it is:** how **people** get set up with AI tools, matched to their role, and how we check they have the right access, no more and no less.

**Who uses it:** new employees, the AI department, and anyone setting up a Mac for agent work.

**Its parts:**

| Part | What it does |
|---|---|
| **Profiles** | 4 access tiers: **user** (uses AI through a UI) → **operator** (runs prepared projects) → **builder** (creates projects) → **engineer** (writes production code, approves PRs). Plus `-phi` versions for people allowed to handle patient data |
| **Tools registry** | One file per approved tool: how to install, permissions, risks (Claude Code, gws, Slack, GitHub, Linear, Figma, AWS…) |
| **Workspace kit** | `setup-workspace.sh`: one script that sets up an operator's or builder's Mac in 20–30 minutes |
| **Patterns** | Credential rules: "1Password to share, Keychain to use, never a plain file" |
| **Skills** | Claude skills: set up a workstation, audit someone's access, schedule tasks (`lst`) |
| **`lst` scheduler** | Runs a command or a Claude session automatically at set times on your Mac |

**How it's used (new operator):**
1. Get access: Claude seat, 1Password vault, Slack channel.
2. Run `bash ~/Downloads/setup-workspace.sh`. It installs Homebrew, Claude Code, Zed, 1Password, `gws` and git, and creates `~/Workspaces`.
3. Import a project zip with `/import-project`.
4. Fill in its secrets with `fill-env`. Set up Google with `setup-google.sh`.
5. Start working. Claude's replies begin with 🌻, which shows it read the workspace rules.

**Example (`lst` scheduler):**
> You want a Slack briefing every morning.
> - `./lst catalog` shows a ready-made `slack-briefing` task.
> - `./lst install slack-briefing` sets it to run at 10:00, 12:00 and 15:30.
> - At those times your Mac opens a terminal, Claude summarizes your Slack channels by topic, and the run is marked done.
> - `./lst verify` confirms it is installed, wakes the Mac, fired on time, and completed.

**Example (access check):**
> You → Claude: *"Check that my Google login doesn't have admin permissions."*
>
> Claude runs `scripts/verify-gws-auth.sh`, which is read-only. Result: `USER-SCOPED, 13 passed, 0 failed`.

**Rules:**
- Least privilege.
- Never widen access to make a task pass; escalate instead.
- Never print secrets.
- **PHI never goes to a standard Claude subscription.**

#### At a glance: kind, data connection and credentials

**Kind:** 🖥️ Setup script + CLI tools · **Data connection:** 🔴 Live (installs + login checks)

**Credentials:**

| Credential | Needed for | From |
|---|---|---|
| **Anthropic (Claude) seat** + its Google group | Claude Code login (the code arrives via the group email) | AI Dept / your lead |
| **1Password** AI Vault / AI Builders | Getting keys and the gws config | AI Dept |
| **`internal-ai-systems` GCP project access** | `gws` login | Google admin |
| **Slack** login (OAuth in `/mcp`) | Slack plugin | Your own Slack account |
| **Linear** (OAuth or API key) | Linear MCP | Your account |
| **GitHub PAT**, approved by an org admin | Builders and engineers only | GitHub admin |
| Mac admin password | Installing software | Your Mac |

---

## 1.5 Investor Materials (`investor_materials/`)

**What it is:** the official **AI story** for investors, the website and sales, plus a map of the April 2026 investor deck showing where AI content should go.

**The 3 AI pillars (`ai_pillars.md`):**
- **P1 Product AI:** a shopping advisor for customers.
- **P2 Operations AI:** submissions, enrollment, maintenance.
- **P3 Engineering AI:** agentic workflows, design-to-code.

Each pillar has a one-line marketing version, concrete achievements, and technical detail.

**Who uses it:** leadership and marketing, when writing investor or sales material.

**Example:**
> You → Claude: *"Suggest AI additions to the investor deck's slide 14 using the P2 pillar, level 2 only."*
>
> Claude reads `presentation_slides.md`, which lists slide 14 as a high-leverage spot, and `ai_pillars.md`. It drafts bullet points that claim **only shipped or in-progress work**.

**Rules:**
- Never claim aspirational work as done.
- Name what the AI does instead of just saying "AI".
- The moat is our data and workflow, not the model.

#### At a glance: kind, data connection and credentials

**Kind:** 📄 Docs · **Data connection:** 🟢 File-based

**Credentials:** none.

---
---

# 2. Sales (`departments/sales/`)

**What the department does:** sales sells Zorro's health benefits (ICHRA) to employers through brokers. Its AI projects speed up **quoting** (turning a census into a quote) and **proposals** (what the prospect sees).

---

## 2.1 Census Ingestion: "Census Raccoon" (`census_ingestion/`)

**What it is:** when a broker sends an employer's **census** (the list of employees and family members), every broker uses a different spreadsheet layout. Census Raccoon turns any layout into **Zorro's standard census spreadsheet**, ready for Tornado, the quoting system.

It's a **Mac app** for account executives (AEs). Under the hood it uses Claude on **AWS Bedrock** (approved for patient data) plus Python code.

**Who uses it:**
- **AEs** use the app.
- **Engineers** improve the rules and code.
- The **admin** sets up AWS access once.

**Status:** app v0.2.0, distributed through Drive. Profile → extract → normalize → validate → output work in the app.

**How it's used (AE):**
1. Download the `.dmg` from the Drive folder "Census Raccoon Releases for AEs" and install it.
2. First launch: paste the Bedrock key from 1Password. It's stored in the Mac Keychain.
3. Click **New enrollment** and drop in the broker's files.
4. Run the 5 steps, one by one or with "autorun":

   | Step | Who does it | Output |
   |---|---|---|
   | 1 Profile | AI | understands the layout → `input_profile.md`. **You review** |
   | 2 Extract | AI | `persons.csv` + `plans.csv` |
   | 3 Normalize | code | `normalized.xlsx` |
   | 4 Validate | code (+ AI fixes) | validation report |
   | 5 Output | AI | `result.md` summary. **You review** |

5. Answer **decision gates** when the app asks.
6. Download `normalized.xlsx` and load it into Tornado.

**Example:**
> An AE gets `Employer-X census.xlsx`: three sheets, with dependents listed in columns to the right of each employee.
> 1. **Profile** detects a "far-right-wide" layout, 42 employees and 3 plan options.
> 2. **Gate 1.5** asks: *"Plan choice appears in both column F and column K. Which is the elected plan?"* The AE picks column K.
> 3. **Extract** produces 42 employees and 37 dependents.
> 4. **Normalize** writes `normalized.xlsx`: DOBs in MM/DD/YYYY, coverage tiers EE/ES/EC/FA.
> 5. **Validate** finds 2 missing ZIP codes and reports them as gaps instead of guessing.
> 6. **Output** writes a summary. The AE uploads the file to Tornado.

**Developer example:**
> You → Claude: *"Run the baseline evaluation on all 4 examples and tell me which scores dropped."*
>
> Claude follows `harness/workflows/evaluate_baseline.md`, runs `diff_checker` against the expected answers, and reports a score per example (Excellent 90+ / Good 70–89 / …).

**Key commands (developer):**
```bash
cd departments/sales/census_ingestion/census_ui
./run.sh --watch             # run the app in dev mode at http://127.0.0.1:5814
pytest                       # run the ~305 tests
```

**Watch out:**
- **Real census files are PHI.** They live only in `asset_processing/` and `baseline_eval/`, which are gitignored.
- Use only fake names in anything committed.
- Never distribute a build tagged `WITH_PHI`.

#### At a glance: kind, data connection and credentials

**Kind:** 🖥️ **Mac app** (AI inside) · **Data connection:** 🟢 File-based; AI calls go live to AWS Bedrock; **no live Tornado link**

**Credentials:**

| Credential | Who | Where it lives |
|---|---|---|
| **AWS Bedrock API key** ("Census Ingestion — Bedrock API Key") | Every AE, on first launch | 1Password → pasted into the app → **macOS Keychain** |
| AWS IAM admin + Bedrock model access | Admin, once (`provision_bedrock_key.sh`) | Admin's AWS login |
| Google (`gws`) with write access to "Census Raccoon Releases for AEs" | Whoever publishes a new DMG | Publisher's Google login |
| GitHub access | Engineers (CI builds) | GitHub |

⚠️ The Bedrock key is **shared and long-lived** today; the deployable plan wants to replace it.

---

## 2.2 Proposals: "Proposal Studio" (`proposals/`)

**What it is:** generates **on-brand sales proposals** for prospects:
- an interactive web page (dark theme)
- a printable PDF (light theme)

It replaced **Qwilr**. AEs use a single offline HTML file, the **Proposal Studio**, which builds a proposal from a **Tornado quote export**.

**Who uses it:**
- **AEs** build proposals.
- **Builders and engineers** maintain the template and rules.
- **Prospects** read the result.

**Status:** template v0.8, Studio **v0.8.31-beta**, distributed through the Drive folder "Proposal Studio for AEs".

**How it's used (AE):**
1. Download `proposal_studio_v0.8.31-beta.html` from Drive and open it in Chrome. No install needed.
2. Pick your AE identity.
3. Drop in the Tornado quote export (JSON) for the prospect.
4. Choose plan designs, confirm fees, edit the copy, and turn optional slides on or off.
5. Download `Zorro Proposal — <Company>.html` and the PDF, and send them to the prospect.

**Example (AE):**
> An AE has a quote for "Acme Logistics" with 3 plan options.
> - They load the quote and choose the "executive" variant.
> - They hide the onboarding-timeline slide and fix one sentence of copy.
> - They download the HTML and PDF and email them.
> - Every fee figure comes from one fee ledger, so the numbers match across slides.

**How it's used (builder): the strict flow**
1. Try ideas in `2_harness/experiments/`. Nothing there ships.
2. Put the decision into the **template** (`2_harness/templates/proposal/v0.8/`).
3. **Regenerate** outputs with the CLI. Never hand-edit `3_publications/`.
4. Run the tests, bump the factory patch number, and publish the new Studio to Drive.

**Example (builder):**
> You → Claude: *"Change request CPY-42: rename 'Monthly allowance' to 'Monthly contribution' everywhere."*
>
> Claude:
> - updates `figures_and_terminology.md` and the template
> - re-renders the sample proposal with `npm run proposal -- render ...`
> - runs `npm test` and shows you the diff against the committed sample
> - bumps `factory_patch`

**Key commands:**
```bash
cd departments/sales/proposals/2_harness/tools/studio
npm install && npm run build && npm test
npm run proposal -- build-studio --version v0.8
```

**Watch out:**
- Template first, always.
- Real prospect data and Tornado dumps are never committed.
- Brand rules: sanctioned icons only; the HTML is dark and the PDF is light.

#### At a glance: kind, data connection and credentials

**Kind:** 🖥️ **Offline HTML app** (one file, no install) · **Data connection:** 🟢 File-based, fully offline; **no live Tornado link**

**Credentials:**

| Credential | Who | Notes |
|---|---|---|
| Google Drive access to "Proposal Studio for AEs" | AEs | To download the Studio |
| Tornado access | AEs | To export the quote JSON |
| `gws` (Google), Figma, Slack MCP | Builders | To read the design and marketing sources |
| None at runtime | — | The Studio runs offline; no keys inside |

---
---

# 3. Operations (`departments/operations/`)

**What the department does:** operations runs day-to-day customer work: enrolling members with insurance carriers, customer support (CREM, Support, Payments), help-center content and hiring for those teams. Its projects reduce manual work and give managers visibility.

---

## 3.1 Carrier Enrollment: "Enrollment Copilot" (`carrier_enrollment/`)

**What it is:** a **Chrome extension** that helps an operator enroll a member on an insurance carrier's website. The operator still does the work, but the extension **pre-fills the form fields** from Zorro's data. The operator **reviews and clicks** instead of typing sensitive data by hand.

**Who uses it:**
- **Ops operators** use the extension.
- **Engineers** build and maintain the "recipes" (per-carrier instructions).

**Status:** proof of concept, extension v0.6.1. First real carrier: **Capital BlueCross**.

**Key idea, the "recipe":** each carrier has a `flow.json` describing its website: the pages, fields and buttons, and which Zorro data goes into each field. **No AI is used while filling**; filling is exact and repeatable.

**How it's used (operator):**
1. Open the dedicated Chrome profile (`make chrome`) and **log in to the carrier site yourself**. The tool never holds carrier passwords.
2. Open the Copilot side panel and load the member's record.
3. On each page, the panel recognizes the page and shows which fields it will fill.
4. Click **Fill**, check the values, click Next. Repeat for each page.
5. **You** click Submit. The result is recorded.

**Example:**
> An operator enrolls "Alex Smith" plus a spouse on Capital BlueCross.
> - On the "Applicant info" page, the panel fills name, DOB, address and SSN from Zorro's record.
> - On the "Dependents" page, it repeats the fill for the spouse.
> - On the "Payment" page, bank details are kept in memory only, never logged.
> - The operator reviews each page and submits.
> - A 30-minute typing job becomes about 10 minutes of reviewing.

**Developer example:**
> You → Claude: *"Check the Capital BlueCross recipe and replay it against the mock site."*
>
> Claude runs:
> - `make recipe-integrity-check CARRIER=capital_bluecross`
> - `make self-replay`, which fills the local fake carrier site at 127.0.0.1:8080 with a synthetic member
> - `make recipe-status`
>
> It reports a score (e.g. 44/75, "still in building").

**Watch out:**
- Never defeat captchas.
- Test data must always be fake: SSNs like `000000XXXX`, `555` phone numbers.
- The mock site is for testing the extension only. The real site always wins.

#### At a glance: kind, data connection and credentials

**Kind:** 🖥️ **Chrome extension** · **Data connection:** 🔴 Live on the carrier website. Live production data pull **not yet cleared** for real use

**Credentials:**

| Credential | Who | Where it lives |
|---|---|---|
| **Carrier portal logins** | Operators (typed by hand; the tool never stores them) | **Zorro Ops 1Password vault** |
| **2FA codes** (some carriers, e.g. CBC) | Operators | Slack `#2fa` channel |
| **Google service-account key** (deposit-only, one Drive folder) | The extension, to upload recordings | **Bundled in the extension**; rotation due 2026-12-02 |
| **AWS Bedrock** | Engineers using the `probe` CLI | AWS login |
| Zorro production data access | The production data pull | Per project docs (not yet cleared for real use) |

⚠️ Recordings contain PHI and are uploaded unredacted (internal only).

---

## 3.2 Unified CRM Cockpit: "Zong" (`unified_crm_cockpit/`)

**What it is:** **one dashboard for all customer teams.** It pulls conversations, tickets and calls from **Intercom** and shows:
- what's still open
- which companies, brokers and carriers have problems
- response times
- issue heat-maps
- AI summaries
- quality checks on CSMs

Today it's a **single HTML file** that opens offline. There's no server.

**Who uses it:**
- **Ops managers** (CREM, Support, Payments) use the dashboard.
- **Engineers / the AI department** run the data pipeline and maintain the rules.

**Status:** v0.14.0, the most active project in the repo.

**How it's used:**
1. **Pull data:** `npm run dump` downloads recent Intercom data. It contains personal data and stays on the machine.
2. **Process:** analyze → enrich (response times etc.) → adapt (a scrubbed version and a real version).
3. **Optional AI:** in the owner's Claude session, generate ticket summaries and issue labels. These are shown with a ✦ mark and never affect numbers.
4. **Build:** `npm run build` produces one HTML file.
5. **Publish:** `publish.ts` uploads it to the restricted Drive folder. Older builds are archived.
6. **Managers** download the HTML and open it in a browser.

**Example (manager):**
> An Ops manager opens the cockpit and:
> - goes to the **Today** tab and sees 12 items still open, 3 of them with no response in over 24 hours
> - filters by broker, and sees that "Broker Agency X" has 5 open coverage issues
> - opens the **Heatmap** tab, and sees that "carrier application status" questions spiked this week

**Example (developer):**
> You → Claude: *"Add a rule that flags a company if its latest enrollment ends in 30 days and there's no upcoming enrollment."*
>
> Claude:
> - adds a structured rule in `src/model/rules/`
> - bumps the rule version and `MODEL_VERSION`, and adds a CHANGELOG line
> - updates the Lineage tab entry and the test
> - runs `npm run model && npm test`, then bumps `VERSION`

**Key commands:**
```bash
cd departments/operations/unified_crm_cockpit
npm run synth && npm run build     # demo build on fake data → prototype/cockpit.html
npm test && npm run model
```

**Watch out:**
- Raw Intercom data never goes in git.
- **AI never decides numbers or flags.**
- Every rule change needs a version bump.

#### At a glance: kind, data connection and credentials

**Kind:** 📊 **Generated report** (offline HTML dashboard) · **Data connection:** 🟡 On-demand pull. The dashboard is a **dated snapshot, not live**

**Credentials:**

| Credential | Who | Where it lives |
|---|---|---|
| **`INTERCOM_API_TOKEN`** (read-only) | Whoever runs the pipeline | `.env` in the project (gitignored) |
| **Google (`gws`)** with access to the "UniCREM Unified CRM Cockpit" Drive folder and the project Doc | Publisher | Your Google login |
| Claude Code session (owner's) | AI summaries on real data | The only approved AI route |
| None for viewers | Managers just open the HTML | Drive folder access only |

---

## 3.3 Customer Teams (`customer_teams/`)

**What it is:** **4 small agent projects run by ops staff themselves** (operators and builders, not engineers) in their own `~/Workspaces`. The folders in agentlab are **snapshots** for visibility. Don't edit them here; change the working copy, then snapshot again.

Each one uses the project standard (section 1.3), its own Google robot account, and the operator's own Slack.

### a) Enrollment Dashboard (`enrollment-dashboard/`)
**What:** builds the **leadership enrollment dashboard**: employees enrolled, lives, carriers, states, industries, participation, financials. It's built from Metabase exports and published as one permanent Drive link.

**Owner:** Bradley Howell. **Audience:** leadership.

**How it's used:**
1. Bradley downloads two Metabase reports.
2. He tells Claude: *"New data is in, build version 12."*
3. Claude creates `assets/v12_<date>/`, runs `harness/tools/run_all.sh` (Python stages that compute every metric with fixed formulas), and builds the HTML from the previous version.
4. Claude checks it renders with no errors and compares it to v11 (e.g. "enrolled employees +4.2%").
5. **Bradley approves**, and Claude overwrites the single live Drive file.

**Rules:**
- Numbers come only from the scripts.
- Publishing needs owner approval.
- No automatic Slack posts.
- No employee-level data on the page.

#### At a glance: kind, data connection and credentials

**Kind:** 🤖 Agent project → 📊 HTML dashboard · **Data connection:** 🟢 File-based (manual downloads); publishing to Drive is live

**Credentials:**

| Credential | Where it lives |
|---|---|
| **Metabase** login (manual downloads) | Bradley's account |
| **Project Google account** `enrollment-dashboard-agent` (keyless) | `oauth2l` sign-in + Token Creator role |
| **`LIVE_DASHBOARD_FILE_ID`** (the file is "anyone with the link", so its ID is treated as a secret) | 1Password → Keychain via `fill-env` |
| Slack (operator's own) | Slack MCP |

### b) Help Center Sync (`help-center-sync/`)
**What:** when a product or process changes, it finds **every help-center article and internal SOP that's now out of date** and proposes exact edits.

**Owner:** Lerone Lessner.

**Example:**
> Lerone: *"The status 'Carrier application sent' is being renamed to 'Enrollment in process' on Nov 1."*
>
> Claude searches the 5 Knowledge Center master docs, the SOPs and the live help.myzorro.co. It writes a **proposal copy** and a **tracker sheet** with one row per finding:
> - "Employees KC §4.2: Rewrite (old: '…application sent', new: '…in process')"
> - "Employers KC §2.1: Screenshots only"
> - "Intercom auto-message: Adjacent target, notify the owner"
>
> A person copies the approved text into Intercom.

**Rules:** never edit the master docs; never publish to Intercom.

#### At a glance: kind, data connection and credentials

**Kind:** 🤖 Agent project → Google Doc + Sheet · **Data connection:** 🔴 Live reads of Google docs and the help site

**Credentials:**
  - **project Google account** `help-center-sync-agent` (keyless), Viewer on the KC and SOP folders, Contributor on the output folder
  - Chrome DevTools (to check the live site and Looms)
  - Slack (own)

### c) ICHRA Takeover Objections (`ichra-takeover-objections/`)
**What:** helps win prospects who are **switching to Zorro from another ICHRA provider**. It turns the competitor's claims and the prospect's questions into accurate, reviewed answers: a transition deck, a Q&A doc, or single replies.

**Owner:** Ann Marie G. **Ops reviewer:** Bradley Howell.

**Example:**
> An AE shares the prospect's question list.
> - Claude extracts each claim, checks it against the objection bank and factual guardrails, and drafts a 6-slide transition deck plus a Q&A doc.
> - Anything uncertain is marked `[NEEDS CONFIRM]`.
> - Ops and the owner both approve, and the **AE sends it**. Claude never contacts anyone outside Zorro.

#### At a glance: kind, data connection and credentials

**Kind:** 🤖 Agent project → deck + Doc · **Data connection:** 🟢 File-based, one folder per deal

**Credentials:** **project Google account** `ichra-objections-agent` (keyless); Slack (own).

### d) CS & Implementation Recruiting (`cs-implementation-recruiting/`)
**What:** screens **job applicants** for Customer Success and Implementation roles and keeps the interview tracker current.

**Owners:** Ann Marie G and Bradley Howell.

**How it's used:** a scheduled task (`lst`) runs 3 times a day. Claude:
1. reads new applications from the careers sheet (read-only)
2. scores each one on domain fit, excellence and benefits experience (US-based is required)
3. appends the results to the screening sheet, skipping duplicates by email
4. sends the team a Slack DM with the top candidates

**People make every hiring decision.** Claude never contacts candidates.

#### At a glance: kind, data connection and credentials

**Kind:** 🤖 Agent project, **scheduled** → Sheets + Slack · **Data connection:** 🔴 Live, on a schedule

**Credentials:**

| Credential | Where it lives |
|---|---|
| **Project Google account** `careers-screening-agent` (keyless): read the careers sheet, write the screening and tracker sheets, read Calendar | `oauth2l` sign-in |
| **HubSpot token** `HUBSPOT_TOKEN` (for resumes; *still being created*) | Keychain via `fill-env` |
| Slack (own) | Slack MCP |

---
---

# 4. Compliance (`departments/compliance/`)

## 4.1 AI Policy (the whole department folder)

**What it is:** where Zorro's **AI and data policies** are written, reviewed and published. Zorro handles patient data and is HIPAA-regulated, so these policies define what staff and builders may do with AI.
- Policies are written in Markdown.
- A script converts them into **branded Google Docs**.
- They're tracked in the **Policy Management** sheet.

**Who uses it:**
- the compliance team and policy owners (CCO, CISO, VP AI Engineering, Operations Manager)
- reviewers who comment in Google Docs
- all staff, who read the published policies

**Status:** 7 policies drafted, none approved yet:

| ID | Policy |
|---|---|
| POL-001 | Data Classification |
| POL-002 | Data Usage |
| POL-003 | AI Data Usage with LLM |
| POL-004 | AI Systems Integrations |
| POL-005 | Google Drive |
| POL-006 | Slack |
| POL-007 | AI Foundations |

**How it's used (lifecycle):**
1. **Write** the draft from the template (`templates/v2.0.0/policy_article.md`).
2. **Register** it in the Policy Management sheet: version, status, owner.
3. **Push** it to Drive:
   ```bash
   python3 tools/push_to_drive.py <file.md> --push
   ```
   This updates the **single draft Google Doc** for that policy.
4. **Review:** reviewers comment in the Doc. Before each new push, the tool **saves all comments** and blocks if any are unresolved.
5. **Approve:** the approver fills in the Decision Log, and the status becomes Approved at v1.0.0.
6. **Publish:**
   ```bash
   python3 tools/push_to_drive.py <file.md> --push --publish
   ```
   This makes a **read-only copy** in Publications and adds one row to the Decision Log.

**Example:**
> You → Claude: *"Fold the resolved review comments into the Slack policy and push the draft."*
>
> Claude:
> - edits `management/drafts/systems/slack.md`
> - runs a dry run, then `--push`
> - the tool harvests the comments into `management/comments/`, sees 2 still unresolved, and stops
> - after you confirm they're addressed, Claude re-runs with `--comments-carried`
> - the same Google Doc is updated, and its ID and comment history are kept

**Most important rule:** **one draft Doc per policy, for its whole life.** Never create a new Doc for a new version, because that loses the comment history. (This happened once with the Slack policy, and it can't be undone.)

**Other rules:**
- The sheet is the source of truth for version and status.
- The Decision Log is append-only.
- Use `Required / Never / Recommended` wording.

#### At a glance: kind, data connection and credentials

**Kind:** 📄 Docs + Python publishing tool · **Data connection:** 🔴 Live sync with Drive/Sheets when run

**Credentials:**
  - **`gws` with your own Google login** (there's no service account here), with edit access to the Policies Drive folders and the Policy Management sheet
  - `pandoc` installed (not a credential)

---
---

# 5. General Admin (`departments/general_admin/`)

## 5.1 Systems Manager (`systems_manager/`)

**What it is:** an internal **web app** that replaces scattered spreadsheets about Zorro's **vendors and software systems**. It shows:
- who owns each tool
- what it costs and when it renews
- which departments use it
- what data it touches
- whether it sends data to AI

**Who uses it:**
- **Admins** (full control)
- **Department users** (record their department's tool usage)
- **Vendor owners** (manage issues for their vendors)

**Status:** MVP, runs locally. Milestones M0–M4 are done. It's planned as the **first app** on the `deployable` hosting platform.

**How it's used:**
1. Sign in with your Zorro Google account.
2. Pick your department.
3. Browse **Vendors** and **Systems**, check the **Dashboard** for upcoming renewals and AI usage, and update **My Department**: "we use Slack for internal data, no PHI, AI features on".

**Example:**
> A compliance officer asks: *"Which systems send Confidential data to an AI provider?"*
>
> In the app, they filter Department Usages by AI usage = yes and classification = Confidential, and get the list in seconds instead of cross-checking 3 spreadsheets.

**How to run it (developer):**
```bash
cd departments/general_admin/systems_manager/app
docker compose up -d                    # database
npm install
cp server/.env.example server/.env      # fill in the Google OAuth client
npm run --workspace server db:migrate && npm run --workspace server db:seed
npm run --workspace server dev          # API on :3000
npm run --workspace client dev          # app on http://localhost:5173
```

**Watch out:**
- New users aren't admins; promote yourself with `db:studio`.
- There are no tests or backups yet (only a plan).

#### At a glance: kind, data connection and credentials

**Kind:** 🖥️ **Web app** (database, **no AI**) · **Data connection:** 🔴 Live database, **manual entry** (no auto-sync)

**Credentials:**

| Credential | Where it lives |
|---|---|
| **Google OAuth client ID + secret** (Zorro-only sign-in) | `server/.env` (gitignored) |
| **`JWT_SECRET`** (login sessions) | `server/.env` |
| **`DATABASE_URL`** (local Postgres, default user locally) | `server/.env` |
| Users: their **Zorro Google account** | — |

Passwords for tracked vendors are **not stored**; it only links to 1Password.

---
---

# 6. Production (`departments/production/`)

**What the department does:** engineering and production. **Most of its AI work moved to the other repo, `ZorroEng/agent-production-hub`.** Only 2 projects live here; for anything else, switch repos.

---

## 6.1 Engineering Support (`engineering_support/`)

**What it is:** the process for **support staff asking engineering for help** with a production issue:
- **T0:** a support person submits a ticket via Slack, or via the OniCall extension.
- **T1:** an AI triage agent classifies and routes it. *Planned, not built yet.*
- **T2:** the on-call engineer gets it in the right channel.

It defines the **intake form**, the **priority tiers and response times**, and **OniCall**, a Chrome extension for reporting bugs with evidence.

**Who uses it:** CS, Ops and support teams (submit), engineers (respond), managers (set the response-time commitments).

**The priority tiers:**

| Tier | Meaning | Response / fix | Where it goes |
|---|---|---|---|
| **P0 Wide** | Many customers affected | 30 min / 4 h, 24/7. **Manager approval needed** | Call the on-call engineer |
| **P1 Scoped** | Some customers blocked | 1 h / 12 h in business windows. Peer check | `#contact-engineering` |
| **P2 Degraded** | Annoying, there's a workaround | 1–2 Israel business days | `#contact-engineering-non-urgent` |
| **Backlog** | Feature request | none | Product requests channel |

**How it's used (submitter, with OniCall):**
1. Install the OniCall extension. Engineers build it with `make install`; it's loaded in Chrome.
2. When something breaks in the Zorro app, open the side panel and click **Record**. Reproduce the problem, then **Stop**. It captures network traffic, page structure, your clicks and console errors.
3. Fill in the 5-step form: what happened, who's affected, and the priority. P0 requires a manager's approval.
4. Submit. The ticket goes to the Slack List, a post appears in the right channel, and the evidence is attached.

**Example:**
> A CS agent sees: "Employer dashboard shows $0 contribution for all employees of one company."
> - They record the page loading, select P1 (one company affected, blocked), and get a peer check.
> - The ticket posts in `#contact-engineering` with the recording.
> - The on-call engineer responds within 1 hour.

**Watch out:** recordings may contain patient data and session cookies. They never go to an AI, and the build zips are kept in a restricted Drive folder.

#### At a glance: kind, data connection and credentials

**Kind:** 📄 Process + 🖥️ **Chrome extension** + Slack form · **Data connection:** 🔴 Live (posts to Slack)

**Credentials:**

| Credential | Who | Where it lives |
|---|---|---|
| **Slack app credentials** (client ID/secret, bot/user tokens, list and channel IDs) | Engineers building OniCall | `local/.env` (gitignored) |
| Each submitter's **own Slack login** (OAuth in the extension) | Support staff | Slack |
| Client secret + limited bot token **embedded in the build** | — | ⚠️ Build zips are kept in a **restricted** Drive folder |
| Drive access to the "OniCall builds" folder | Whoever releases | Google login |

---

## 6.2 Local Setup (`local_setup/`)

**What it is:** a step-by-step guide for running **Zorro's production code** on your machine:
- `zorro-ts`: TypeScript monolith + app
- `zorro-py`: Python agent hub

Claude walks you through it.

**Who uses it:** engineers setting up a new machine.

**How it's used:**
> You → Claude (in agentlab): *"Production → local_setup. Walk me through setting up zorro-ts and zorro-py."*

Claude follows `execution/local-dev-setup.md` step by step:
1. Homebrew.
2. Clone both repos.
3. Node, pnpm and nx.
4. Python via uv.
5. AWS CLI and Docker.
6. `.env` files: these come as an **encrypted zip from Nadav**, with the password sent separately.
7. Start the database.
8. Reset it locally.
9. Start the servers:

| Service | Port |
|---|---|
| Agent Hub | 8000 |
| Decision Backoffice | 3000 |
| Monolith API | 3333 |
| App | 3009 |

You run the login and password steps yourself.

*This is directly useful for you, since you have `zorro-ts` open. The guide was tested on an Apple Silicon Mac, so expect differences on Windows.*

#### At a glance: kind, data connection and credentials

**Kind:** 📄 Guide (Claude walks you through) · **Data connection:** 🔴 Live (GitHub, AWS)

**Credentials:**

| Credential | Where it comes from |
|---|---|
| **GitHub** access to `ZorroEng` repos | GitHub org |
| **AWS** login (`aws login`) + Kubernetes access (`akl`) | Your AWS account |
| **`.env` files** for both repos | **Encrypted zip from Nadav Kaufman**, password sent separately |
| Docker | Installed locally (not a credential) |

---
---

# 7. Cross (`departments/cross/`)

**What the department is:** not a real department, but a home for **projects that serve several departments**.

## 7.1 Zorro Deck (`zorro_deck/`)

**What it is:** a system for building **Zorro presentations with agents**.
- You write the deck once as **data**: a JSON file describing each slide's content and layout.
- The system turns it into **HTML** (quick to review), **PowerPoint**, and **Google Slides** (for collaboration).
- All three look the same and come from the same source.

**Who uses it:** anyone building a Zorro deck with Claude. The first real use is the **investor deck** (45 slides, confidential).

**Status:** v0.3.4. The 84-slide Zorro template gallery is rebuilt in HTML, and the investor deck is migrated.

**How it's used:**
1. **Pull** an existing deck from Google Slides (read-only): `pull_deck.py`.
2. Describe each slide in `spec.json`: its archetype (cover, agenda, cards, chart…), content and theme. **The content holds data only**; the colours and fonts come from the design system.
3. **Build:** `npm run build -- projects/<deck>` produces `deck.html` and `deck.pptx`.
4. **Review:** `npm run review` compares against the original and checks for overflow and overlap.
5. **Publish:** `npm run publish -- projects/<deck>` creates or updates the Google Slides copy. Republishing keeps the same link.

**Example:**
> You → Claude: *"In the investor deck, update slide 12's KPI numbers: ARR $X, 120 employers, 35 states. Rebuild and show me the review."*
>
> Claude edits that slide's content in `spec.json`, runs `npm run build` and `npm run review -- --only 12`, and shows the HTML side-by-side with the original. After you approve, it runs `npm run publish`.

**Watch out:**
- Never edit the Google Slides copy as if it were the source; pull edits back into the spec.
- Investor deck content is never committed.
- Confirm with the owner before writing to Drive.

#### At a glance: kind, data connection and credentials

**Kind:** 🖥️ **CLI tool** (TypeScript) → 📊 slides · **Data connection:** 🟢 File-based build; publish to Slides on demand

**Credentials:**
  - **`gws` with your Google login**, with access to the source deck and the target Drive folder (e.g. the InvestorDeck shared drive)
  - Google Chrome (for screenshots and review)

---
---

# 8. Knowledge Management and Marketing

- **`knowledge_management/`:** only a "Moved" note. The department and its KB templates now live in **ZorroEng/agent-production-hub**.
- **Marketing:** listed in the root `AGENTS.md` but **reserved**. There is no folder and no projects yet.

---
---

# 9. Deployable (`deployable/`), a root-level project

**What it is:** a **plan** (no code yet) for one shared platform to host agentlab's apps properly on Zorro's AWS infrastructure, instead of running them on laptops or Drive. It would have a **test** and a **production** environment.
- **Internal apps** (for staff): Google sign-in.
- **External apps** (for brokers, clients, prospects): Frontegg login and/or secure per-link tokens.

**Who uses it:** the AI department and engineering/platform teams, for planning.

**Status:** design only (Sept 2026). Nothing moves until the design is reviewed.

**Planned order (migration phases):**

| Phase | App |
|---|---|
| 0 | Empty platform + two "hello" apps (one internal, one external) |
| 1 | **Systems Manager** (pilot) |
| 2 | **Extension server**: auto-updates for Chrome extensions (Enrollment Copilot, OniCall) |
| 3 | Carrier registry API |
| 4 | OniCall evidence service (removes the Slack secret from the extension) |
| 5 | Hosted Census Raccoon (needs legal sign-off for PHI) |
| 6 | Proposals: Studio for AEs (internal) + hosted proposal links for prospects (external) |

**Example (what it would enable):**
> Today an AE emails a proposal PDF.
>
> After phase 6, they'd click **Publish** in the hosted Studio and get a secure link for the prospect. The link can expire or be revoked, and its views are tracked. The proposal is stored safely in S3 and never contains employee data.

**How you'd use it now:** read `.agents/design.md` and `.agents/decisions.md`. The open questions (Q1–Q10) need answers from the platform, compliance and product owners before any building starts.

#### At a glance: kind, data connection and credentials

**Kind:** 📄 Design doc · **Data connection:** —

**Credentials:** none now. The plan uses AWS (EKS, Secrets Manager, IAM roles per app), a Google OAuth client per environment, Frontegg, and signing keys in AWS KMS, with **no static secrets** in apps.

---
---


---

# Part 3: Where credentials live

| Credential type | Projects | Stored in |
|---|---|---|
| Your own Google login (`gws`) | AI Org Deck, Compliance, CRM Cockpit (publish), Zorro Deck, publishing DMGs | Keychain + encrypted gws config |
| Project Google account (keyless) | The 4 customer-team projects, AI Projects Setup | `oauth2l` sign-in + Token Creator role |
| AWS Bedrock key | Census Raccoon | 1Password → Keychain |
| Intercom token | CRM Cockpit | project `.env` |
| Slack app secrets | OniCall | `local/.env` |
| Carrier logins | Enrollment Copilot | Ops 1Password vault (typed by hand) |
| HubSpot token, dashboard file ID | Recruiting, Enrollment Dashboard | 1Password → Keychain (`fill-env`) |
| Google OAuth client + JWT secret | Systems Manager | `server/.env` |
| Production `.env` files | Local Setup | Encrypted zip from Nadav |
| Claude seat, Slack, Linear, GitHub PAT | Everyone, depending on tier | Your own accounts |


---

# Part 4: Procedures by role

*Assumes a Mac unless a step says otherwise. Where a step says "ask", it is a person's decision, not something to work around.*

## Which role am I right now?

| I want to... | Role | Section |
|---|---|---|
| Understand the repo or a project, with no changes | Learner | 1 |
| Use a finished tool (app, dashboard, proposal) | End user | 2 |
| Run an existing agent project's tasks | Operator | 3 |
| Create a new agent project in my workspace | Builder | 4 |
| Change code, rules or docs in agentlab | Contributor | 5 |
| Add a brand-new project or initiative to agentlab | Contributor + AI Dept | 6 |
| Review and merge someone's PR | Reviewer (engineer tier only) | 7 |
| Hand a project to operators, or bring their copy back | AI Dept | 8 |
| Publish a result to Google Drive | Contributor / Operator | 9 |

One person can hold several roles. Pick the role for **the task in front of you**.

---

## 0. Rules for every role, every time

1. **Scope first:** tell Claude the department and project before anything else.
2. **Stay in that project's folder.** Ask before touching anything else.
3. **No patient or personal data (PHI/PII) in git, chat or Slack.** Use fake names (Employer A, Alex Smith, EMP001).
4. **No secrets in files or chat.** Keys go in 1Password, the Keychain, or a gitignored `.env`.
5. **Never contact outsiders** (clients, brokers, candidates) through the agent: no email, invites or sharing.
6. **Never push to `main`.** Use a branch and a PR.
7. **When unsure, stop and ask** in Slack `#maya-ai-projects`.

**Emergency:** if a secret or real personal data was committed, pasted or shared, **stop, don't try to hide it, and tell the AI department immediately.** Secrets must be rotated, and git history may need a rewrite.

---

## 1. Learner: understand the repo or a project (works on Windows too)

**Needs:** the repo and Claude Code. Nothing else.

1. Get the latest code:
   ```bash
   cd agentlab
   git checkout main
   git pull
   ```
2. Open the folder in VS Code or Zed, then open Claude Code.
3. Start the conversation:
   > Read the root AGENTS.md. Department: `<dept>`, project: `<project>`. Read its AGENTS.md and explain what it does, how it works, and how to run it. **Don't change anything.**
4. Go deeper by asking: "walk me through the `harness/` folder", "what does `<script>` do", or "what's on the roadmap in `.agents/`".
5. Read these yourself in order: root `AGENTS.md`, then the project's `AGENTS.md`, then its `.agents/` roadmap.

**Done when** you can explain the project's purpose, inputs, outputs and rules in your own words.

---

## 2. End user: use a finished tool

**Needs:** nothing from the repo. The project team gives you the tool.

| Tool | How you get it | How you use it |
|---|---|---|
| Census Raccoon (sales AEs) | `.dmg` installer from the census team. An admin sets up AWS access once (see `sales/census_ingestion/census_ui/how_to_run.md`) | Install the app, drop in a broker census file, get the standard Zorro census |
| CRM Cockpit (ops managers) | HTML file in the restricted Drive folder "UniCREM Unified CRM Cockpit" | Download it, open it in a browser, use the tabs |
| Sales proposals (prospects) | Sent by the sales rep | Open the HTML or PDF |
| AI policies (all staff) | Google Docs in the Compliance › Publications folder | Read |

**Problems or feedback:** tell the project owner (named in the project's `AGENTS.md` → Team) or the project's Slack channel. Don't edit the output yourself.

---

## 3. Operator: run an existing agent project (in ~/Workspaces, no git or PRs)

### One-time setup
1. Get access: a Claude seat (and its Google group, for shared seats), the **1Password AI Builders** vault, and your project's Slack channel.
2. Get `setup-workspace.sh` from the AI department (Drive → AI Projects – Limited Access → Workspace kit).
3. Open Terminal and run it:
   ```bash
   bash ~/Downloads/setup-workspace.sh           # 20–30 min, safe to re-run
   bash ~/Downloads/setup-workspace.sh --check   # verify; changes nothing
   ```
4. Finish the manual steps it lists: sign in to 1Password, log in with `claude`, run `/mcp` for Slack, then log out and back in.
5. Get your project as a **zip** from the AI department (Drive → AI Projects → Project zips).
6. Open Zed on `~/Workspaces`, start Claude, and type:
   ```
   /import-project ~/Downloads/<project>.zip
   ```
7. If the project needs secrets, run:
   ```bash
   ~/Workspaces/.shared/bin/fill-env <project>
   ```
8. Set up Google access for the project:
   ```bash
   ~/Workspaces/.shared/connectors/google/setup-google.sh <project>
   ```

### Each work session
1. Open Zed on `~/Workspaces` and start Claude. Its first reply should start with 🌻, which shows it read the rules.
2. Say: "Let's work on `<project>`", then give the task in plain English.
3. If Claude says secrets are more than 7 days old, run `fill-env <project>` again.
4. **Answer decision questions yourself.** Claude stops on judgment calls by design.
5. Review the output before anyone else sees it. **Claude drafts; you approve.**
6. Save progress: ask Claude to "commit this to git" (local workspace git only).

**Never:** run `gws auth login`, edit `.shared/`, or ask Claude to email or share with outsiders.

---

## 4. Builder: create a new agent project in your workspace

1. Do the operator one-time setup (section 3, steps 1–4).
2. Make sure the idea went through the **AI Initiative Shop**: it should be submitted in the AI Initiative Candidates doc and approved (see section 6, step 1).
3. Copy the standard empty project:
   ```bash
   cp -R ~/Workspaces/.shared/project_standard/template ~/Workspaces/<project-name>
   ```
4. Ask Claude to help fill in every `<…>` in `AGENTS.md`, `HANDOFF.md` and `connectors/`. Include `## Team` and `## External boundary`.
5. Ask the AI department for the project's **Google service account**. They create it and share the needed folders. Put its name in `connectors/google/connector.conf`.
6. Secrets: list them in `.env.example` as 1Password references, then run `fill-env <project>`.
7. Verify Google access:
   ```bash
   ~/Workspaces/.shared/connectors/google/setup-google.sh <project>
   ```
8. Put each kind of content where it belongs:
   - task rules → `harness/rules/`
   - procedures → `harness/skills/`
   - code → `harness/tools/`
   - learnings → `.agents/invariants_and_learnings.md`
   - each run → `.agents/run_log.md`
9. Tell the AI department when it's ready. They'll checkpoint it into agentlab (section 8).

---

## 5. Contributor: update something in agentlab

### One-time setup
1. Access: GitHub `ZorroEng` org (and your token approved by an admin), Claude seat, 1Password, Drive folders for your project.
2. Install:
   ```bash
   brew install git gh googleworkspace-cli 1password-cli node python
   ```
   Then install Claude Code, plus Zed or VS Code.
3. Log in:
   ```bash
   gh auth login
   claude
   ```
   For Google, use `gws auth login` only if your profile allows a personal login, and tick only the scopes you need.
4. Clone the repo:
   ```bash
   git clone https://github.com/ZorroEng/agentlab.git
   ```

### Each change
1. **Start from the latest `main` and make a branch:**
   ```bash
   cd agentlab
   git checkout main
   git pull
   git checkout -b <department>/<short-topic>
   ```
   For example: `operations/cockpit-new-chart`.
2. **Set up the project** if it's your first time, following its `AGENTS.md`:
   - TypeScript projects: `npm install`
   - Python projects: `pip install -r requirements.txt`
   - Then `cp .env.example .env` and fill the values from 1Password.
3. **Open Claude in `agentlab/` and scope it:**
   > Read the root AGENTS.md. Department: `<dept>`, project: `<project>`. Read its AGENTS.md. Task: `<what you want>`. Show me a plan before changing anything.
4. **Review the plan**, then let Claude do it. Watch the diffs.
5. **Follow the project's own rules.** For example, the cockpit requires:
   - a `VERSION` bump in the same commit
   - a rule version bump plus a CHANGELOG line when a model rule changes
   - a matching Lineage entry update
6. **Run the project's checks.** For example `npm run typecheck`, `npm test`, `npm run build`, or the project's test script.
7. **Check what you're about to commit:**
   ```bash
   git status
   git diff
   ```
   Confirm there's no `.env`, no raw data, no real names, and nothing outside your project folder.
8. **Commit** using the repo's message style:
   ```bash
   git add <specific files>
   git commit -m "feat(<department>/<project>): <what changed>"
   ```
   Types: `feat` (new), `fix` (bug), `docs` (docs only).
9. **Push and open a PR:**
   ```bash
   git push -u origin <branch>
   gh pr create
   ```
   In the description, say what changed, why, and how you tested it.
10. **Get a review** from an engineer-tier reviewer. Fix any comments, then they merge.
11. **After the merge**, if the project publishes to Drive, publish the new version (section 9).

---

## 6. Add a brand-new project or initiative to agentlab

1. **Submit the idea** in the AI Initiative Candidates doc, on your department's tab, with: pain + flow, an example case, and a quick evaluation (size, severity, rate, timing). Tag your department lead.
2. **Wait for evaluation.** The AI department and your department lead score it on owner, support tier, data exposure, fallback and systems path. **If it touches PHI, Compliance must clear it first.**
3. **If approved:** owners are named and a proof-of-concept time-box is set (2–6 weeks).
4. **Ask before creating folders.** Adding a module changes the root `AGENTS.md` and department routing, and those files are shared. Get approval from the AI department.
5. Create `departments/<dept>/<project>/` with an `AGENTS.md` as the entry point, plus `.agents/` for plans. Follow the standard shape in `ai_dept/ai_projects_setup/project_standard/`.
6. Add a `.gitignore` for any raw or PHI folders **before** any data arrives.
7. Follow section 5 (branch → PR → review) for everything.

---

## 7. Reviewer: review and merge a PR (engineer tier only)

1. Fetch the PR locally:
   ```bash
   gh pr checkout <number>
   ```
2. Check its **scope**: does it only touch the declared project? Any change to the root `AGENTS.md`, `.claude/`, `.github/` or another department needs explicit approval.
3. Check **data and secrets**: no PHI/PII, real names, keys or `.env` files.
4. Check **project rules**: version bumps, changelog, tests passing.
5. Run the project's tests yourself.
6. Optionally, ask Claude to review the PR.
7. Approve and merge on GitHub, or leave comments.

---

## 8. AI department: hand projects to operators and bring them back

**Hand-off:**
1. Build the project zip from the working copy (secrets are never included).
2. Put it in Drive → AI Projects → Project zips.
3. The operator imports it with `/import-project` (section 3).

**Checkpoint back into agentlab** (for example `operations/customer_teams/<project>`):
1. Copy the project from the working copy, leaving out everything its `.gitignore` excludes.
2. Reapply the redactions and scan for personal names and secrets.
3. Commit via a branch and PR (section 5).

Never edit checkpoint folders directly. Change the working copy first, then checkpoint again.

---

## 9. Publish a result to Google Drive

1. Use the project's own publish command from its `AGENTS.md`. Don't upload by hand. For example:
   ```bash
   node src/publish/publish.ts prototype/dist/<file>.html --dry-run    # cockpit: dry run first
   ```
2. Check the dry-run output: which folder, which file, and whether anything older gets archived.
3. Run it for real.
4. Share links only inside Zorro, never with outsiders, unless the project owner explicitly asks.
5. Record it where the project says to: the run log, the roadmap, or the decision log for compliance.

---

## Quick reference: the opening message to Claude

```
Read the root AGENTS.md first.
Department: <dept>    Project: <project>
Read that project's AGENTS.md.
Task: <what you want>.
<"Don't change anything" | "Show me a plan first">
```


---

## Quick reference: which project, for which need?

| I need to… | Project |
|---|---|
| propose a new AI idea | AI Initiative Shop |
| set up my Mac for agent work | AI Workstation Setup (workspace kit) |
| give an agent Google access | AI Projects Setup (Google connector) |
| turn a broker census into the standard format | Census Ingestion (Census Raccoon) |
| make a sales proposal | Proposals (Proposal Studio) |
| fill carrier enrollment forms faster | Carrier Enrollment (Enrollment Copilot) |
| see open customer issues across teams | Unified CRM Cockpit |
| update the leadership enrollment dashboard | Customer Teams → enrollment-dashboard |
| find stale help-center articles | Customer Teams → help-center-sync |
| answer a competitor's claims to a prospect | Customer Teams → ichra-takeover-objections |
| screen job applicants | Customer Teams → cs-implementation-recruiting |
| write or publish an AI policy | Compliance |
| find who owns a software tool | General Admin → Systems Manager |
| report a production bug to engineering | Production → Engineering Support (OniCall) |
| run zorro-ts / zorro-py locally | Production → Local Setup |
| build a Zorro slide deck | Cross → Zorro Deck |
| plan hosting for an app | Deployable |
