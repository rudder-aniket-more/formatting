# agentlab: department and project guide

*A plain-language guide to every department and project in the repo: what it is, who uses it, how it's used, and a worked example. As of Oct 2026.*

> **About the examples.** The examples show a realistic session or task based on how each project's docs describe its use. All names, companies and numbers in them are made up. In the "You → Claude" lines, Claude has already been scoped to the project (see "How every project is used" below).

---

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

---
---

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

### c) ICHRA Takeover Objections (`ichra-takeover-objections/`)
**What:** helps win prospects who are **switching to Zorro from another ICHRA provider**. It turns the competitor's claims and the prospect's questions into accurate, reviewed answers: a transition deck, a Q&A doc, or single replies.

**Owner:** Ann Marie G. **Ops reviewer:** Bradley Howell.

**Example:**
> An AE shares the prospect's question list.
> - Claude extracts each claim, checks it against the objection bank and factual guardrails, and drafts a 6-slide transition deck plus a Q&A doc.
> - Anything uncertain is marked `[NEEDS CONFIRM]`.
> - Ops and the owner both approve, and the **AE sends it**. Claude never contacts anyone outside Zorro.

### d) CS & Implementation Recruiting (`cs-implementation-recruiting/`)
**What:** screens **job applicants** for Customer Success and Implementation roles and keeps the interview tracker current.

**Owners:** Ann Marie G and Bradley Howell.

**How it's used:** a scheduled task (`lst`) runs 3 times a day. Claude:
1. reads new applications from the careers sheet (read-only)
2. scores each one on domain fit, excellence and benefits experience (US-based is required)
3. appends the results to the screening sheet, skipping duplicates by email
4. sends the team a Slack DM with the top candidates

**People make every hiring decision.** Claude never contacts candidates.

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

---
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
