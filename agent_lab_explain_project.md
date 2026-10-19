# agentlab: every project, its output, and the credentials it needs

*Based on each project's own docs (Oct 2026). "Credentials" means logins, keys, tokens or access grants. **Never put any of them in the repo.** They live in 1Password, the macOS Keychain, or a gitignored `.env`.*

---

## Overview

| # | Project | Department | Used by | Output | Credentials? |
|---|---|---|---|---|---|
| 1 | AI Initiative Shop | AI Dept | All departments, AI Dept | Process docs + exec deck (PPTX/PDF) | Google Doc access only |
| 2 | AI Org Deck | AI Dept | AI Dept → leadership | Slide decks (PPTX/PDF) | Google access to rebuild |
| 3 | AI Projects Setup | AI Dept | AI Dept, builders, agents | A Google service account per project | **Yes**: GCP admin, 1Password, oauth2l |
| 4 | AI Workstation Setup | AI Dept | New staff, AI Dept | A configured Mac + `~/Workspaces`; scheduled tasks | **Yes**: Claude seat, Google, 1Password, Slack… |
| 5 | Investor Materials | AI Dept | Leadership, marketing | Narrative docs + investor deck | No |
| 6 | Census Raccoon | Sales | AEs (users), engineers | **Excel file** (`normalized.xlsx`) + summary; the app is a Mac `.dmg` | **Yes**: AWS Bedrock key |
| 7 | Proposal Studio | Sales | AEs → prospects | **Interactive HTML + PDF proposal** | AEs: Drive + Tornado access. Builders: Google, Figma, Slack |
| 8 | Enrollment Copilot | Operations | Ops operators | **Completed enrollment on the carrier website** + recordings in Drive | **Yes**: carrier logins, AWS, Google service account |
| 9 | CRM Cockpit | Operations | Ops managers | **Single HTML dashboard** + PDF reports + Doc tab | **Yes**: Intercom API token, Google |
| 10 | Enrollment Dashboard | Operations (team) | Bradley → leadership | **HTML dashboard** at one permanent Drive link | **Yes**: Metabase, project Google account, 1Password |
| 11 | Help Center Sync | Operations (team) | Lerone (KC owner) | **Google Doc proposal + tracker Google Sheet** | **Yes**: project Google account |
| 12 | ICHRA Takeover Objections | Operations (team) | Ann Marie, AEs, Ops | **Deck (HTML→PDF), Q&A Google Doc, reply text** | **Yes**: project Google account |
| 13 | CS Recruiting | Operations (team) | Ann Marie, Bradley | **Rows in Google Sheets + Slack DM** | **Yes**: project Google account, HubSpot token, Slack |
| 14 | AI Policy | Compliance | Compliance, policy owners, all staff | **Google Docs** (drafts and published) + Decision Log **sheet** | **Yes**: your own Google login + Drive folder access |
| 15 | Systems Manager | General Admin | Admins, department users | **Web app** (database) | **Yes**: Google OAuth client, database, JWT secret |
| 16 | Engineering Support / OniCall | Production | Support staff, engineers | **Slack ticket + channel post + evidence files**; Chrome extension | **Yes**: Slack app credentials |
| 17 | Local Setup | Production | Engineers | **zorro-ts / zorro-py running locally** | **Yes**: GitHub, AWS, encrypted `.env` files |
| 18 | Zorro Deck | Cross | Deck builders (e.g. investor deck) | **HTML + PowerPoint + Google Slides** | **Yes**: Google (`gws`) + Drive folder access |
| 19 | Deployable | Root | AI Dept, platform team | **Design document only** (future: hosted apps) | Not yet |

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
