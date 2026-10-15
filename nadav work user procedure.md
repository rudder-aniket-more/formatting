# agentlab: step-by-step procedures by role

Based on the repo's own docs (Oct 2026). Assumes a Mac unless a step says otherwise.
Where a step says "ask", it is a person's decision, not something to work around.

---

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
