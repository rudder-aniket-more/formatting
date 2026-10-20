# agentlab: description and talking points

For use with `agentlab-summary.html`. For details on any project, see `agentlab-guide.md`.

---

## Short description (to send with the file)

> **agentlab: summary**
>
> agentlab is Zorro's repository for AI agent projects across departments. Each project solves one business problem, such as turning broker census files into our standard format or giving ops one dashboard of customer issues. An AI agent (Claude) does the work by following written rules, and people review and approve the results. This page covers what it is, how it works, the main projects, who uses it, the safety rules, strengths and open risks.

**One-line version:**
> agentlab is where Zorro builds AI agents that do real work for each department, under clear rules and with people in control.

---

## Talking points (about 5 minutes)

### 1. What it is (30 sec)
- agentlab is one GitHub repository for **AI agent projects across all departments**.
- Each project solves **one specific business problem**.
- It's less a traditional codebase and more a **set of instructions for an AI agent**, plus the scripts it uses.
- **Key message:** the AI does the work, people stay in control.

### 2. How it works (1 min)
- **It starts with a real problem.** A department submits it, and the AI department checks five things: who owns it, what support it needs, how sensitive the data is, what happens if it breaks, and how it will be built.
- **Approved ideas become a project** with written rules, an `AGENTS.md` file the agent reads first.
- **To use it,** someone opens Claude, names the department and project, and asks in plain English.
- **Claude follows the rules,** runs the scripts, and **stops to ask a person** at decision points.
- **Results go to the right place:** code to GitHub, documents to Google Drive, sensitive data stays on the local machine.

### 3. Main projects (1.5 min)
Pick 2–3 to highlight, depending on your audience.
- **Sales, Census Raccoon:** broker census files come in every format; this Mac app turns them into our standard format. *In use.*
- **Sales, Proposal Studio:** builds branded proposals from a quote. It replaced Qwilr. *Beta.*
- **Operations, CRM Cockpit:** one dashboard of open customer issues across Intercom, by company, broker and carrier. *Prototype, the most active project.*
- **Operations, Enrollment Copilot:** pre-fills carrier enrollment forms, so operators review instead of retyping sensitive data. *Proof of concept.*
- **Operations, team-run projects:** ops staff run their own agents for the leadership dashboard, help-center updates, competitor objections and recruiting screening.
- **Also:** compliance policies, a vendor tracker, an engineering bug reporter, a deck builder, and a hosting plan.

### 4. Who uses it (45 sec)
- The repo defines four access tiers, ordered by authority:
  - **user:** everyday AI through a UI
  - **operator:** runs harnesses set up for them, adapts rules and settings, reviews outputs
  - **builder:** creates projects and writes their rules and procedures
  - **engineer:** changes and reviews production code, merges and approves PRs
- Each person gets the **lowest level that lets them do their job**.
- Many people only use the result, such as the app, dashboard or proposal, without touching the repo. *(This is a description, not one of the repo's tiers.)*

### 5. Safety (45 sec)
Stress this part.
- **Patient data never goes into the repository.**
- Sensitive AI work runs **only through approved channels** (AWS Bedrock).
- Agents **cannot email, message or share** with anyone outside Zorro, and the tooling enforces this.
- Each agent has **its own limited Google identity**, never a person's login.
- Secrets live in 1Password or the Keychain, never in files.

### 6. Strengths and open risks (45 sec)
- **Strengths:** one consistent pattern for every project, governance built in from the start, non-engineers can run agent projects, and real tools are already in use.
- **Risks:**
  - most of the work depends on one person
  - setup and tools assume macOS
  - most apps aren't hosted yet
  - the AI policies are still drafts, and some patient-data flows await compliance decisions

### 7. Current state and next step (30 sec)
- Several tools are **already used** by sales and operations.
- Most apps still **run locally or from Google Drive**.
- **Next:** "Deployable", a shared hosting platform on Zorro's AWS for the most-used apps.
- **Decisions needed:** which apps to host first, a second owner for key projects, and whether Windows support is needed.
- **Questions:** Slack `#maya-ai-projects`.

---

## Likely questions and short answers

| Question | Answer |
|---|---|
| Is this replacing people? | No. Agents prepare and draft; people decide, approve and send. |
| Is it safe with patient data? | PHI stays out of the repo, and sensitive AI work uses only approved AWS Bedrock paths. Agents can't contact outsiders. |
| Who maintains it? | Mainly the AI department, with department owners for each project. |
| How does a team get a project? | Submit the problem in the AI Initiative Candidates doc; the AI department evaluates it. |
| Can anyone use it? | Access depends on role. Most people use the finished tools; fewer people run or build projects. |
| What's the biggest risk? | Most of the work depends on a small team, and most apps aren't yet on a proper hosting platform. |
| What do you need from us? | *(Fill in: e.g. which projects to prioritize, access decisions, your expected role.)* |
