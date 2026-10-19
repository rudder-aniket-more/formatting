# agentlab: every project, its output, and the credentials it needs

*Based on each project's own docs (Oct 2026). "Credentials" means logins, keys, tokens or access grants. **Never put any of them in the repo.** They live in 1Password, the macOS Keychain, or a gitignored `.env`.*

---

## Overview: the full picture

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

| # | Project · Dept · Kind | What it does · Who uses it | How it's used (input → steps → output) | Data connection | Credentials |
|---|---|---|---|---|---|
| 1 | **AI Initiative Shop**<br>AI Dept<br>📄 Process / docs | The approval route for new AI ideas: decides owner, support level, data risk, fallback and how to build.<br>**Users:** any employee submits; dept leads + AI Dept evaluate | **Input:** problem + example + quick evaluation, by any employee, in the "AI Initiative Candidates" Google Doc.<br>**Steps:** submit → scored on 5 points → PHI goes to Compliance first → time-boxed proof of concept → release (needs runbook + support contact) → maintain.<br>**Output:** a go/no-go decision; approved ideas become a project | 🟢 Manual (Google Doc) | Zorro Google account |
| 2 | **AI Org Deck**<br>AI Dept<br>📊 Generated report (slides) | Exec decks explaining Zorro's AI approach.<br>**Users:** AI Dept presents to leadership and teams | **Input:** slide text in `deck.md` (AI Dept) + initiatives from the Candidates Doc.<br>**Steps:** edit text → `python build_deck.py`.<br>**Output:** **PowerPoint + PDF** | 🟡 Pulls the Doc on rebuild | Your Google login (`gws`) |
| 3 | **AI Projects Setup**<br>AI Dept<br>📄 Process + setup scripts | Gives each agent project **its own Google robot account** with access to specific folders only; defines the standard project folder layout.<br>**Users:** AI Dept (creates), builders (request), agents (use) | **Input:** project name, needed Drive folders, who may use it, from the builder.<br>**Steps:** AI Dept creates the account → shares the folders (Viewer by default) → grants the person → person runs `setup-google.sh <project>`.<br>**Output:** a working, verified **Google identity for that project**; `gws` inside the project acts as it | 🔴 Live (each Google call gets a fresh 15-min token) | GCP admin (AI Dept), 1Password, your `oauth2l` sign-in |
| 4 | **AI Workstation Setup**<br>AI Dept<br>🖥️ Setup script + CLI tools | Sets up a **person's Mac** with the right AI tools for their role and proves the access is correct. Includes `lst`, the task scheduler.<br>**Users:** new staff, AI Dept | **Input:** the person's role/tier + their logins and approvals.<br>**Steps:** request access → run `setup-workspace.sh` (or `install.sh --profile <tier>`) → do the manual logins → run the verify script → optionally `lst install <task>`.<br>**Output:** a **configured Mac** with `~/Workspaces`, a verification report, scheduled tasks | 🔴 Live (installs + login checks) | Claude seat, Google group, 1Password, GCP project access, Slack, (GitHub PAT for builders/engineers) |
| 5 | **Investor Materials**<br>AI Dept<br>📄 Docs | The official AI story for investors and sales.<br>**Users:** leadership, marketing | **Input:** AI achievements + the investor PPTX, from leadership/AI Dept.<br>**Steps:** read the pillars and slide map → write AI content into the deck.<br>**Output:** updated investor **deck / text** | 🟢 File-based | None |
| 6 | **Census Raccoon**<br>Sales<br>🖥️ **Mac app** (AI inside) | Converts any broker's employee census spreadsheet into **Zorro's standard census format** for quoting.<br>**Users:** **AEs**; engineers maintain it | **Input:** broker census files (xlsx/csv/pdf), dropped in by the **AE**.<br>**Steps:** install `.dmg` from Drive → paste Bedrock key once → New enrollment → drop files → Profile (AI) → **AE reviews** → Extract (AI) → Normalize (code) → Validate (code) → Output (AI) → **AE reviews** → answer decision questions along the way.<br>**Output:** **Excel file `normalized.xlsx`** (Employees + Plans) + summary → AE uploads it to Tornado | 🟢 File-based; AI calls go live to AWS Bedrock; **no live Tornado link** | **AWS Bedrock key** (1Password → Keychain) |
| 7 | **Proposal Studio**<br>Sales<br>🖥️ **Offline HTML app** (one file, no install) | Builds **branded sales proposals** from a quote. It replaced Qwilr.<br>**Users:** **AEs** build them; **prospects** read them; builders maintain the template | **Input:** Tornado quote export (JSON) + AE's choices, by the **AE**.<br>**Steps:** download Studio HTML from Drive → open in Chrome → pick your AE identity → load the quote → choose plans, confirm fees, edit text, toggle slides → download.<br>**Output:** **interactive HTML proposal + PDF** → AE emails them to the prospect | 🟢 File-based, fully offline; **no live Tornado link** | AEs: Drive + Tornado access. No keys inside the app |
| 8 | **Enrollment Copilot**<br>Operations<br>🖥️ **Chrome extension** | **Pre-fills insurance carriers' enrollment forms** from Zorro data, so operators review instead of retyping sensitive data.<br>**Users:** **Ops operators**; engineers write per-carrier "recipes" | **Input:** the member's enrollment record (loaded by the **operator**) + the carrier's recipe (by **engineers**).<br>**Steps:** open the dedicated Chrome profile → log in to the carrier site yourself → open the side panel → load member → on each page click **Fill**, check, Next → **operator submits**.<br>**Output:** **completed enrollment on the carrier's website** + session recording saved to Drive | 🔴 Live on the carrier website. Live production data pull **not yet cleared** for real use | Carrier logins (Ops 1Password, typed by hand), 2FA via Slack, Google robot key bundled in the extension |
| 9 | **CRM Cockpit ("Zong")**<br>Operations<br>📊 **Generated report** (offline HTML dashboard) | **One dashboard of customer issues** from Intercom: open items, response times, problem companies/brokers/carriers, AI summaries, CSM quality checks.<br>**Users:** **Ops managers** view; engineers/AI Dept build | **Input:** Intercom conversations, tickets, calls; the pull is run by an **engineer / AI Dept**.<br>**Steps:** `npm run pack` (pull → analyze → enrich → adapt → optional AI summaries → build) → `publish` to Drive → managers download and open in a browser.<br>**Output:** **one HTML dashboard file** (+ PDF reports, Gap Report Doc tab) | 🟡 On-demand pull. The dashboard is a **dated snapshot, not live** | Intercom API token (`.env`), Google login to publish |
| 10 | **Enrollment Dashboard**<br>Operations (team)<br>🤖 Agent project → 📊 HTML dashboard | Builds the **leadership enrollment dashboard**: employees, lives, carriers, states, money.<br>**Users:** **Bradley** runs it; **leadership** views it | **Input:** 2 Metabase exports, downloaded by **Bradley**.<br>**Steps:** tell Claude "new data, build next version" → scripts compute every number → Claude builds the HTML and compares with the last version → **Bradley approves** → Claude overwrites the live Drive file.<br>**Output:** **HTML dashboard at one permanent Drive link** | 🟢 File-based (manual downloads); publishing to Drive is live | Metabase, project Google account, file ID in 1Password |
| 11 | **Help Center Sync**<br>Operations (team)<br>🤖 Agent project → Google Doc + Sheet | Finds **help-center articles and SOPs that a change has made outdated**, and proposes exact edits.<br>**Users:** **Lerone** (Knowledge Center owner) | **Input:** change description, type, effective date (+ PRD/Loom/Slack links), from **Lerone** or the change owner.<br>**Steps:** tell Claude the change → it reads all KC docs, SOPs and the live help site → writes findings (Remove / Rewrite / Screenshots only / New needed / Flag…) → a person copies approved text into Intercom.<br>**Output:** **proposal Google Doc + tracker Google Sheet** | 🔴 Live reads of Google docs and the help site | Project Google account |
| 12 | **ICHRA Takeover Objections**<br>Operations (team)<br>🤖 Agent project → deck + Doc | Prepares **accurate answers for prospects switching from a competitor** to Zorro.<br>**Users:** **Ann Marie** (owner), **Bradley** (Ops reviewer), the deal's **AE** | **Input:** the prospect's questions, the competitor's claims, census counts, per deal, from the **AE / Ann Marie**.<br>**Steps:** Claude matches claims against the approved answer bank and fact rules → drafts → unsure items marked `[NEEDS CONFIRM]` → **Ops + owner approve** → **AE sends**.<br>**Output:** **transition deck (PDF), Q&A Google Doc, or reply text** | 🟢 File-based, one folder per deal | Project Google account |
| 13 | **CS Recruiting**<br>Operations (team)<br>🤖 Agent project, **scheduled** → Sheets + Slack | **Screens job applicants** for CS/Implementation roles and keeps the interview tracker current.<br>**Users:** **Ann Marie, Bradley** | **Input:** applications arrive in the careers sheet from the website form (by **candidates**, automatic); interviews come from Calendar.<br>**Steps:** runs automatically 3×/day → reads new applicants → scores fit (US-based required) → adds rows → updates the tracker → DMs top candidates to the team. **People make every hiring decision.**<br>**Output:** **rows in Google Sheets + Slack DM** | 🔴 Live, on a schedule | Project Google account, HubSpot token (in progress), Slack |
| 14 | **AI Policy**<br>Compliance<br>📄 Docs + Python publishing tool | Writes and publishes Zorro's **AI and data policies**, with an approval record.<br>**Users:** compliance team and policy owners write and approve; all staff read | **Input:** policy text (Markdown) by **compliance / owners**; register details in the sheet; reviewers' comments.<br>**Steps:** write draft → `push_to_drive.py --push` (updates the one draft Doc, saves comments) → reviewers comment → owner approves in the sheet → `--push --publish`.<br>**Output:** **Google Docs** (living draft + read-only published copy) + **Decision Log rows in a Google Sheet** | 🔴 Live sync with Drive/Sheets when run | Your own Google login + edit access to the Policies folders |
| 15 | **Systems Manager**<br>General Admin<br>🖥️ **Web app** (database, **no AI**) | Replaces spreadsheets: **who owns each software tool, its cost, renewal, which departments use it, what data it touches, whether it uses AI**.<br>**Users:** admins, department users, vendor owners | **Input:** vendors, systems, costs, usage, **typed into forms** by admins and department users.<br>**Steps:** sign in with Zorro Google → pick department → browse/search → record "my department uses X with Y data" → check the dashboard.<br>**Output:** a **web app** to search and filter (runs locally today) | 🔴 Live database, **manual entry** (no auto-sync) | Google OAuth client, JWT secret, database URL (`server/.env`) |
| 16 | **Engineering Support + OniCall**<br>Production<br>📄 Process + 🖥️ **Chrome extension** + Slack form | How support staff **report production problems to engineering**, with priorities P0–P2 and response times. **OniCall** records the bug as evidence.<br>**Users:** CS/Ops/support report; on-call engineers respond | **Input:** a bug report + evidence captured from the browser, by **support staff**.<br>**Steps:** open OniCall → Record → reproduce → Stop → fill the 5-step form, pick priority (P0 needs a manager) → Submit.<br>**Output:** **ticket in a Slack List + post in the right Slack channel + evidence files** | 🔴 Live (posts to Slack) | Slack app credentials (`local/.env`), users' own Slack |
| 17 | **Local Setup**<br>Production<br>📄 Guide (Claude walks you through) | Gets **Zorro's production code (zorro-ts, zorro-py)** running on your own machine.<br>**Users:** engineers | **Input:** GitHub repos, `.env` zip, AWS login, from the **engineer** (+ Nadav sends the env zip).<br>**Steps:** ask Claude for local_setup → install tools → clone → decrypt `.env` → start DB → reset → `aws login` → start the 4 services.<br>**Output:** **app and APIs running locally** (ports 8000, 3000, 3333, 3009) | 🔴 Live (GitHub, AWS) | GitHub, AWS, encrypted `.env` zip |
| 18 | **Zorro Deck**<br>Cross<br>🖥️ **CLI tool** (TypeScript) → 📊 slides | Builds **Zorro presentations from one source file** so the HTML, PowerPoint and Google Slides versions always match.<br>**Users:** deck builders (first: investor deck) | **Input:** slide content in `spec.json` (+ images, existing deck) by the **deck builder**.<br>**Steps:** pull the existing deck → edit `spec.json` → `npm run build` → `npm run review` → `npm run publish`.<br>**Output:** **HTML + PowerPoint + Google Slides** (same link on re-publish) | 🟢 File-based build; publish to Slides on demand | Your Google login + Drive folder access |
| 19 | **Deployable**<br>Root<br>📄 Design doc | **Plan** to host agentlab apps on Zorro's AWS (test + prod), with Google sign-in for staff and secure links for clients.<br>**Users:** AI Dept, platform team | **Input:** decisions from platform, compliance and product owners.<br>**Steps:** answer 10 open questions → build platform → move apps one by one (Systems Manager first).<br>**Output:** today **design docs only**; later **hosted web apps** | — | None yet |

---

# AI Dept

## 1. AI Initiative Shop
- **Use:** the process for turning a department's problem into a supported AI tool.
- **Who uses it:** anyone with an idea (submits); department leads and the AI department (evaluate).
- **Output:** process documents (lifecycle, criteria, risks) and an **exec deck (PPTX + PDF)**. New initiatives become **rows in the "AI Initiative Candidates" Google Doc**.
- **How it works:**
  1. Submit the problem in the Google Doc.
  2. It's scored on 5 points: owner, support tier, data sensitivity, fallback, build path.
  3. Proof of concept.
  4. Release.
  5. Maintain.
- **Credentials:**
  - Your Zorro Google account, with access to the Candidates Doc.
  - To rebuild the deck: Python with python-pptx. No secrets.

## 2. AI Org Deck
- **Use:** explains Zorro's AI approach to leadership and teams.
- **Who uses it:** the AI department presents; leadership and other departments are the audience.
- **Output:** **PPTX + PDF decks** ("Building an AI-Ready Organization", "AI Across Zorro").
- **How it works:** edit `deck.md` → run `build_deck.py` → new PPTX and PDF. Slide 3 is refreshed from the Candidates Doc via `gws`.
- **Credentials:** `gws` with your Google login (to read the Doc). Python with python-pptx.

## 3. AI Projects Setup
- **Use:** gives each **agent project its own Google identity** with minimal access, and defines the standard project folder shape.
- **Who uses it:** the AI department (creates accounts), builders (request them, set up projects), agents (follow the rules).
- **Output:** a **Google service account** per project (`<name>@internal-ai-systems.iam.gserviceaccount.com`), shared only on specific Drive folders, plus a verified `connector.conf` in the project.
- **How it works:**
  1. An admin creates the account.
  2. The admin shares the needed folders with it (Viewer by default) and grants the person permission to use it.
  3. The person runs `setup-google.sh <project>`.
  4. From then on, `gws` inside the project acts as that account. Gmail and sharing changes are blocked.

| Credential | Who needs it | Where it lives |
|---|---|---|
| GCP admin access to the `internal-ai-systems` project (`gcloud`) | AI Dept admin, to create accounts | Admin's own login |
| 1Password CLI, **AI Vault / AI Builders** vault | Admin (stores keys); users (keyless sign-in app) | 1Password |
| **Keyless (preferred):** your `oauth2l` sign-in + the "Token Creator" role on that account | Each person running the project | `~/.config/oauth2l/` (your own sign-in) |
| **Fallback:** the service-account key | Each person | macOS Keychain (from 1Password) |

## 4. AI Workstation Setup
- **Use:** sets up **people's Macs** with the right AI tools for their role, and verifies their access.
- **Who uses it:** new employees, the AI department, anyone running agent projects.
- **Output:**
  - a **configured Mac**: Claude Code, Zed, `gws`, 1Password, git, `~/Workspaces` with rules
  - **verification reports**
  - optional **scheduled tasks** (`lst`), e.g. a Slack briefing several times a day
- **How it works:** run `setup-workspace.sh` (operators and builders) or `install.sh --profile <tier>`, finish the manual logins, then run the verify scripts.

| Credential | Needed for | From |
|---|---|---|
| **Anthropic (Claude) seat** + its Google group | Claude Code login (the code arrives via the group email) | AI Dept / your lead |
| **1Password** AI Vault / AI Builders | Getting keys and the gws config | AI Dept |
| **`internal-ai-systems` GCP project access** | `gws` login | Google admin |
| **Slack** login (OAuth in `/mcp`) | Slack plugin | Your own Slack account |
| **Linear** (OAuth or API key) | Linear MCP | Your account |
| **GitHub PAT**, approved by an org admin | Builders and engineers only | GitHub admin |
| Mac admin password | Installing software | Your Mac |

## 5. Investor Materials
- **Use:** the official AI story for investors, the website and sales.
- **Who uses it:** leadership and marketing.
- **Output:** **Markdown narrative** (3 AI pillars) and a **slide map** for the investor **PPTX**.
- **How it works:** read `ai_pillars.md` and `presentation_slides.md`, then write AI content into the deck.
- **Credentials:** none.

---

# Sales

## 6. Census Raccoon (`census_ingestion`)
- **Use:** converts any broker census spreadsheet into **Zorro's standard census format** for quoting.
- **Who uses it:** **AEs** use the Mac app; engineers maintain the rules and code; an admin sets up AWS once.
- **Output:**
  - **`normalized.xlsx`** (Employees and Plans sheets), ready for Tornado
  - a `result.md` summary and a validation report
  - the app itself is distributed as a **`.dmg`** from Drive
- **How it works:** AE drops in files → AI profiles them → AI extracts people and plans → code normalizes → code validates → AI writes the summary. A person reviews twice and answers decision gates.

| Credential | Who | Where it lives |
|---|---|---|
| **AWS Bedrock API key** ("Census Ingestion — Bedrock API Key") | Every AE, on first launch | 1Password → pasted into the app → **macOS Keychain** |
| AWS IAM admin + Bedrock model access | Admin, once (`provision_bedrock_key.sh`) | Admin's AWS login |
| Google (`gws`) with write access to "Census Raccoon Releases for AEs" | Whoever publishes a new DMG | Publisher's Google login |
| GitHub access | Engineers (CI builds) | GitHub |

⚠️ The Bedrock key is **shared and long-lived** today; the deployable plan wants to replace it.

## 7. Proposal Studio (`proposals`)
- **Use:** creates **branded sales proposals** for prospects. It replaced Qwilr.
- **Who uses it:** **AEs** build proposals; builders maintain the template; **prospects** read them.
- **Output:**
  - **interactive HTML proposal** (dark) + **PDF** (light), named `Zorro Proposal — <Company>`
  - the Studio itself is **one offline HTML file**
- **How it works:**
  1. The AE opens the Studio file in Chrome.
  2. Picks their identity.
  3. Loads the Tornado quote export.
  4. Chooses plans, confirms fees, edits copy.
  5. Downloads the HTML and PDF.
  6. Sends them to the prospect.

| Credential | Who | Notes |
|---|---|---|
| Google Drive access to "Proposal Studio for AEs" | AEs | To download the Studio |
| Tornado access | AEs | To export the quote JSON |
| `gws` (Google), Figma, Slack MCP | Builders | To read the design and marketing sources |
| None at runtime | — | The Studio runs offline; no keys inside |

---

# Operations

## 8. Enrollment Copilot (`carrier_enrollment`)
- **Use:** a **Chrome extension** that **pre-fills carrier enrollment forms** from Zorro data. The operator reviews instead of typing.
- **Who uses it:** **ops operators**; engineers build the per-carrier recipes.
- **Output:**
  - a **completed enrollment on the carrier's website** (the operator submits)
  - **recordings** of the session uploaded to Drive
  - the extension is distributed as a **zip**
- **How it works:**
  1. The operator logs in to the carrier site **manually** in a dedicated Chrome profile.
  2. The side panel recognizes the page and loads the member record.
  3. **Fill** puts data in the fields with no AI involved.
  4. The operator reviews and submits.

| Credential | Who | Where it lives |
|---|---|---|
| **Carrier portal logins** | Operators (typed by hand; the tool never stores them) | **Zorro Ops 1Password vault** |
| **2FA codes** (some carriers, e.g. CBC) | Operators | Slack `#2fa` channel |
| **Google service-account key** (deposit-only, one Drive folder) | The extension, to upload recordings | **Bundled in the extension**; rotation due 2026-12-02 |
| **AWS Bedrock** | Engineers using the `probe` CLI | AWS login |
| Zorro production data access | The production data pull | Per project docs (not yet cleared for real use) |

⚠️ Recordings contain PHI and are uploaded unredacted (internal only).

## 9. CRM Cockpit, "Zong" (`unified_crm_cockpit`)
- **Use:** **one dashboard** of customer issues across Intercom, by company, broker, carrier and employee, with response times and quality checks.
- **Who uses it:** **ops managers** (CREM, Support, Payments) view it; engineers run the pipeline.
- **Output:**
  - **one offline HTML file**, published to a restricted Drive folder
  - **PDF reports** per admin or broker
  - the **Gap Report** written into a Google Doc tab
- **How it works:** pull Intercom data → analyze → enrich → adapt (scrubbed or real) → optional AI summaries → build HTML → publish.

| Credential | Who | Where it lives |
|---|---|---|
| **`INTERCOM_API_TOKEN`** (read-only) | Whoever runs the pipeline | `.env` in the project (gitignored) |
| **Google (`gws`)** with access to the "UniCREM Unified CRM Cockpit" Drive folder and the project Doc | Publisher | Your Google login |
| Claude Code session (owner's) | AI summaries on real data | The only approved AI route |
| None for viewers | Managers just open the HTML | Drive folder access only |

## 10. Enrollment Dashboard (`customer_teams/enrollment-dashboard`)
- **Use:** builds the **leadership enrollment dashboard** (employees, lives, carriers, states, financials).
- **Who uses it:** Bradley Howell runs it; leadership views it.
- **Output:** an **HTML dashboard** that overwrites **one permanent Drive file** (same link every time).
- **How it works:** Bradley downloads 2 Metabase reports → Claude runs the Python metric scripts → builds the HTML → compares with the last version → Bradley approves → it's published.

| Credential | Where it lives |
|---|---|
| **Metabase** login (manual downloads) | Bradley's account |
| **Project Google account** `enrollment-dashboard-agent` (keyless) | `oauth2l` sign-in + Token Creator role |
| **`LIVE_DASHBOARD_FILE_ID`** (the file is "anyone with the link", so its ID is treated as a secret) | 1Password → Keychain via `fill-env` |
| Slack (operator's own) | Slack MCP |

## 11. Help Center Sync (`customer_teams/help-center-sync`)
- **Use:** finds **help-center articles and SOPs that are out of date** after a change, and proposes edits.
- **Who uses it:** Lerone Lessner.
- **Output:** a **proposal copy (Google Doc)** + a **tracker Google Sheet** with one row per finding.
- **How it works:** describe the change → Claude searches the 5 Knowledge Center docs, the SOPs and the live help site → writes findings. A person updates Intercom.
- **Credentials:**
  - **project Google account** `help-center-sync-agent` (keyless), Viewer on the KC and SOP folders, Contributor on the output folder
  - Chrome DevTools (to check the live site and Looms)
  - Slack (own)

## 12. ICHRA Takeover Objections (`customer_teams/ichra-takeover-objections`)
- **Use:** answers to prospects **switching to Zorro from a competitor**.
- **Who uses it:** Ann Marie G (owner), Bradley (Ops reviewer), the deal's AE (sends it).
- **Output:** a **transition deck (HTML → PDF)**, a **Q&A Google Doc**, or **reply text**.
- **How it works:** intake the prospect's questions → match against the objection bank and fact rules → draft → Ops and owner approve → the AE sends.
- **Credentials:** **project Google account** `ichra-objections-agent` (keyless); Slack (own).

## 13. CS & Implementation Recruiting (`customer_teams/cs-implementation-recruiting`)
- **Use:** **screens job applicants** and keeps the interview tracker current.
- **Who uses it:** Ann Marie G and Bradley Howell.
- **Output:**
  - **new rows in the Screening Candidates Google Sheet**
  - **updates to the Interview Tracker sheet**
  - a **Slack DM** with the top candidates
  - resumes archived in Drive
- **How it works:** a scheduled task (`lst`) runs 3×/day → reads new applications (read-only) → scores them → appends to the sheet → DMs the team.

| Credential | Where it lives |
|---|---|
| **Project Google account** `careers-screening-agent` (keyless): read the careers sheet, write the screening and tracker sheets, read Calendar | `oauth2l` sign-in |
| **HubSpot token** `HUBSPOT_TOKEN` (for resumes; *still being created*) | Keychain via `fill-env` |
| Slack (own) | Slack MCP |

---

# Compliance

## 14. AI Policy (`compliance`)
- **Use:** writes and publishes Zorro's **AI and data policies**.
- **Who uses it:** the compliance team and policy owners (writing and approving); all staff (reading).
- **Output:**
  - **Google Docs**: one living draft per policy, plus read-only published copies
  - **Decision Log rows** in the Policy Management **Google Sheet**
  - comment archives in the repo
- **How it works:** write Markdown → `push_to_drive.py --push` (updates the draft Doc, saves comments) → reviewers comment → approver signs off in the sheet → `--publish` makes a read-only copy and logs it.
- **Credentials:**
  - **`gws` with your own Google login** (there's no service account here), with edit access to the Policies Drive folders and the Policy Management sheet
  - `pandoc` installed (not a credential)

---

# General Admin

## 15. Systems Manager (`general_admin/systems_manager`)
- **Use:** tracks **vendors, systems, costs, renewals, data use and AI use**, instead of spreadsheets. **It uses no AI**; it's a regular database app.
- **Who uses it:** admins (everything), department users (their department's usage), vendor owners.
- **Output:** a **web app** (React + API + PostgreSQL). It currently runs locally only.
- **How it works:** users sign in with Zorro Google → browse and search vendors and systems → record usage → view the dashboard.

| Credential | Where it lives |
|---|---|
| **Google OAuth client ID + secret** (Zorro-only sign-in) | `server/.env` (gitignored) |
| **`JWT_SECRET`** (login sessions) | `server/.env` |
| **`DATABASE_URL`** (local Postgres, default user locally) | `server/.env` |
| Users: their **Zorro Google account** | — |

Passwords for tracked vendors are **not stored**; it only links to 1Password.

---

# Production

## 16. Engineering Support + OniCall (`production/engineering_support`)
- **Use:** how support teams **report production problems to engineering**, with priority tiers and response times. **OniCall** is the Chrome extension for reporting with evidence.
- **Who uses it:** CS, Ops and support staff (report); on-call engineers (respond); managers (P0 approval).
- **Output:**
  - a **ticket in a Slack List** and a **post in the right Slack channel**, with **evidence files** (recording, errors, screenshots)
  - training docs and the policy
- **How it works:** in OniCall, record the problem → fill the 5-step form and choose a priority → it posts to Slack and attaches the evidence.

| Credential | Who | Where it lives |
|---|---|---|
| **Slack app credentials** (client ID/secret, bot/user tokens, list and channel IDs) | Engineers building OniCall | `local/.env` (gitignored) |
| Each submitter's **own Slack login** (OAuth in the extension) | Support staff | Slack |
| Client secret + limited bot token **embedded in the build** | — | ⚠️ Build zips are kept in a **restricted** Drive folder |
| Drive access to the "OniCall builds" folder | Whoever releases | Google login |

## 17. Local Setup (`production/local_setup`)
- **Use:** a guide for running **Zorro's production code** (`zorro-ts`, `zorro-py`) on your own machine.
- **Who uses it:** engineers.
- **Output:** **running local services**:

| Service | Port |
|---|---|
| Agent Hub | 8000 |
| Decision Backoffice | 3000 |
| Monolith API | 3333 |
| App | 3009 |

- **How it works:** Claude walks you through `local-dev-setup.md` step by step. You do the login steps yourself.

| Credential | Where it comes from |
|---|---|
| **GitHub** access to `ZorroEng` repos | GitHub org |
| **AWS** login (`aws login`) + Kubernetes access (`akl`) | Your AWS account |
| **`.env` files** for both repos | **Encrypted zip from Nadav Kaufman**, password sent separately |
| Docker | Installed locally (not a credential) |

---

# Cross

## 18. Zorro Deck (`cross/zorro_deck`)
- **Use:** builds **Zorro presentations** from one structured source, so the HTML, PowerPoint and Google Slides versions always match.
- **Who uses it:** anyone building a Zorro deck with Claude; first use is the **investor deck** (confidential).
- **Output:** **`deck.html` + `deck.pptx` + a Google Slides copy** in Drive (same link on re-publish).
- **How it works:** pull the existing deck → describe slides in `spec.json` → `npm run build` → `npm run review` → `npm run publish`.
- **Credentials:**
  - **`gws` with your Google login**, with access to the source deck and the target Drive folder (e.g. the InvestorDeck shared drive)
  - Google Chrome (for screenshots and review)

---

# Root

## 19. Deployable
- **Use:** a **plan** to host agentlab's apps on Zorro's AWS (test and prod).
- **Who uses it:** the AI department and the platform/engineering teams.
- **Output:** today, **design documents only**. Later: hosted web apps (Systems Manager first).
- **Credentials:** none now. The plan uses AWS (EKS, Secrets Manager, IAM roles per app), a Google OAuth client per environment, Frontegg, and signing keys in AWS KMS, with **no static secrets** in apps.

---

## Where credentials live

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
