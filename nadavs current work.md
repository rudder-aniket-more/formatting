# agentlab: briefing notes

*The repo at a glance. Facts are taken from the repo as of 15 Oct 2026.*

## 1. What it is, in one line
agentlab is Zorro's monorepo for **AI agent projects across the company**. Each department brings a pain point, and the AI department turns it into a project that an AI agent (Claude Code) runs under written rules, with scripts and safety guardrails.

## 2. Key facts
- GitHub: `ZorroEng/agentlab`. About 1,770 files and 2,000+ commits, from Feb 2026 to Oct 2026.
- Mostly built by the AI department (Nadav Kaufman). Other contributors: Sarit Bartov, Gadi Jacobson, Maya Cohen Tor.
- Areas: Sales, Operations, Compliance, AI Dept, General Admin, Production, Cross-department, plus a planned "deployable" platform.
- Some production work moved out to a sister repo, `ZorroEng/agent-production-hub`.

## 3. The core idea: the instructions are the product
Most of the repo is **instructions for an AI agent**, not conventional code:
- **`AGENTS.md`** in every folder is the agent's operating manual for that project: scope, rules, steps, and what is forbidden.
- **`harness/`** holds `workflows/` (what to do), `rules/` (what correct output looks like) and `tools/` (small scripts the agent runs).
- **`.agents/`** holds plans, roadmaps and decision logs.
- **Google Drive is the official store** for documents and decisions. The repo holds code and agent notes. Agents read and write Drive through the `gws` command-line tool.

## 4. How it is used day to day
1. A person opens Claude Code in the repo or in their project folder.
2. The agent asks which department and project, then loads that project's `AGENTS.md`.
3. The person asks in plain English, for example "process this census file" or "rebuild the cockpit on the latest data".
4. The agent follows the written workflow, runs the scripts, and publishes the result to Google Drive.

## 5. Main projects

| Department | Project | What it does | Stage |
|---|---|---|---|
| Sales | Census Ingestion ("Census Raccoon") | Turns brokers' messy employee census spreadsheets into Zorro's standard template. Packaged as a Mac app for account executives | Working tool, built automatically as a Mac installer (.dmg) |
| Sales | Proposals | Generates interactive HTML and PDF sales proposals from one template. Built to replace Qwilr, whose contract ended 15 Aug 2026 | Beta releases (v0.8.x); most active project by commits |
| Operations | Unified CRM Cockpit | One dashboard for customer teams built from Intercom data: issues, service levels, broker and carrier gaps, AI summaries, CSM quality checks | Working prototype, v0.14, published to a restricted Drive folder |
| Operations | Carrier Enrollment | Helps operators enroll members on carrier websites: pre-fills forms so people review instead of typing patient data | Design and proof-of-concept stage |
| Operations | Customer Teams projects | 4 small agent projects run by ops staff: enrollment dashboard, help-center sync, ICHRA takeover objections, CS recruiting screening | Running on team members' machines; copies kept here |
| Compliance | AI Policy | Workforce and builder AI policies, plus policies for Slack and Drive. Published to Google Docs with an append-only decision log | Drafts and published policies |
| AI Dept | AI Initiative Shop | The intake process from department pain point to proof of concept to supported tool, scored on owner, support level, data sensitivity, fallback plan and build platform | Defined process with an exec deck |
| AI Dept | Workstation and project setup | One-script Mac setup for non-engineers. Role-based access profiles. A separate Google service account per agent project | In use (setup kit v0.3) |
| General Admin | Systems Manager | Internal app that replaces the vendor, systems and data-usage spreadsheets | MVP |
| Cross-department | Zorro Deck | Write each deck once, then render it to HTML, PowerPoint and Google Slides | v0.2 |
| Root project | Deployable | Plan for one hosting platform for all agentlab apps, with Google sign-in for staff and token links for brokers and clients | Design only |

## 6. Example walkthrough: Census Ingestion
- **Problem:** every broker sends employee census data in a different spreadsheet layout, and reformatting it by hand is slow and error-prone.
- **How it works:** the agent profiles the file, extracts people and plans, and writes the standard Zorro census spreadsheet. Files go through 3 stages: raw → intermediate → final.
- **Quality:** 4 sample employers with known correct answers serve as the test set. Rules cover field mappings and "decision gates", where the agent must stop and ask a human.
- **Delivery:** packaged as the Census Raccoon Mac app, rebuilt automatically by GitHub Actions on every change.

## 7. Safety and governance (the part most worth stressing)
- **Patient and personal data never goes into git.** Real files are excluded by `.gitignore`, and the repo uses made-up names (Employer A, Alex Smith, EMP001).
- **Agents never act on people outside Zorro:** no sending email, no calendar invites, no file sharing, no posting messages. This is enforced by tooling, not only written down.
- **Each agent gets its own Google service account,** limited to specific folders. It never uses a person's login.
- **Secrets stay out of files.** They live in 1Password or the macOS Keychain and are passed to a single command only when needed.
- **Agents stay inside their project folder.** Editing another department's area requires explicit approval.
- **Audit trail:** versioned releases, append-only policy decision logs, and versioned business rules in the cockpit.

## 8. Strengths
- One consistent pattern across every department, so a new project starts from a template.
- Governance is built in from the start rather than added later.
- Non-engineers can run agent projects through the setup kit.
- Real working tools: Census Raccoon, the CRM Cockpit, Proposals.

## 9. Risks and open points
- **Depends on one person:** about 95% of commits come from one author.
- **Runs on Macs only:** setup, Keychain and scheduling all assume macOS.
- **Mostly prototypes:** most apps run locally or from Drive. The "deployable" hosting platform is still a design.
- **Work is split across two repos** (agentlab and agent-production-hub), and some docs are out of date.
- **Unattended AI processing of patient data** (for example scheduled jobs) still needs its own approved route. Today it is approved only through interactive sessions.
- **Wide permissions:** the repo's Claude settings let the agent run commands and edit files without asking.

## 10. Suggested next steps / questions for the boss
- Which projects should move to production first through the "deployable" platform?
- Should a second owner be added for key modules to reduce the single-person risk?
- Is Windows support needed for staff who don't use Macs?
- What is our role or contribution expected to be in this repo?
