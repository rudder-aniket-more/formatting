<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>agentlab: summary</title>
<style>
  body { font-family: Arial, Helvetica, sans-serif; font-size: 14px; line-height: 1.45; color: #222; max-width: 820px; margin: 30px auto; padding: 0 20px; }
  h1 { font-size: 22px; margin: 0 0 4px; }
  h2 { font-size: 16px; margin: 18px 0 6px; border-bottom: 1px solid #ccc; padding-bottom: 3px; }
  p { margin: 6px 0; }
  ul, ol { margin: 4px 0; padding-left: 20px; }
  li { margin: 2px 0; }
  table { border-collapse: collapse; width: 100%; margin: 6px 0; }
  th, td { border: 1px solid #ccc; padding: 4px 7px; text-align: left; vertical-align: top; }
  th { background: #f2f2f2; }
  .sub { color: #666; margin-bottom: 10px; }
  .two { display: flex; gap: 24px; }
  .two > div { flex: 1; }
  .foot { margin-top: 18px; padding-top: 8px; border-top: 1px solid #ccc; color: #555; font-size: 13px; }
  @media print { body { margin: 0; font-size: 12px; } h2 { margin-top: 12px; } }
  @media (max-width: 700px) { .two { flex-direction: column; gap: 0; } }
</style>
</head>
<body>

<h1>agentlab: summary</h1>
<p class="sub">Repository: github.com/ZorroEng/agentlab · October 2026</p>

<h2>What it is</h2>
<p>agentlab is Zorro's repository for <b>AI agent projects across departments</b>. Each project solves one business problem, for example reformatting broker census files or building an ops dashboard. An AI agent (Claude Code) does the work by following written rules and running the project's scripts. People review and approve the results.</p>

<h2>How it works</h2>
<ol>
  <li>A department submits a problem. The AI department checks who owns it, the support level, how sensitive the data is, the fallback, and how it will be built.</li>
  <li>Approved ideas become a project folder with rules (<code>AGENTS.md</code>), procedures and tools.</li>
  <li>A user opens Claude, names the department and project, and gives a task in plain English.</li>
  <li>Claude follows the project rules, stops at decision points, and produces the result.</li>
  <li>Code goes to GitHub, documents and outputs go to Google Drive, and sensitive data stays on the local machine.</li>
</ol>

<h2>Main projects</h2>
<table>
  <tr><th>Department</th><th>Project</th><th>What it is</th><th>Status</th></tr>
  <tr><td>Sales</td><td>Census Raccoon</td><td><b>Mac app</b>: turns broker census spreadsheets into Zorro's standard Excel format</td><td>In use</td></tr>
  <tr><td>Sales</td><td>Proposal Studio</td><td><b>Offline HTML app</b>: builds branded HTML/PDF sales proposals from a quote</td><td>Beta v0.8</td></tr>
  <tr><td>Operations</td><td>CRM Cockpit</td><td><b>HTML dashboard</b> of customer issues, built from Intercom data</td><td>Prototype v0.14</td></tr>
  <tr><td>Operations</td><td>Enrollment Copilot</td><td><b>Chrome extension</b> that pre-fills carrier enrollment forms</td><td>Proof of concept</td></tr>
  <tr><td>Operations</td><td>Customer team projects</td><td><b>Agent projects</b>: enrollment dashboard, help-center updates, competitor objections, recruiting screening</td><td>In use</td></tr>
  <tr><td>Compliance</td><td>AI policies</td><td><b>Policies + publishing tool</b>: AI and data policies as Google Docs</td><td>Drafts</td></tr>
  <tr><td>AI Dept</td><td>Setup and process</td><td>AI project intake, role-based access, Mac setup kit, agent identities</td><td>Active</td></tr>
  <tr><td>Admin</td><td>Systems Manager</td><td><b>Web app</b> tracking vendors, systems, costs and data use</td><td>MVP</td></tr>
  <tr><td>Production</td><td>Engineering Support</td><td>Support-to-engineering process + <b>OniCall</b> bug-report extension</td><td>In progress</td></tr>
  <tr><td>Cross</td><td>Zorro Deck</td><td><b>CLI tool</b>: one deck source → HTML, PowerPoint and Google Slides</td><td>v0.3</td></tr>
  <tr><td>Root</td><td>Deployable</td><td><b>Design</b> for hosting these apps on Zorro's AWS</td><td>Design only</td></tr>
</table>

<h2>Who uses it</h2>
<p>The repo defines four access tiers, ordered by authority:</p>
<ul>
  <li><b>User:</b> uses AI for everyday work, through a UI.</li>
  <li><b>Operator:</b> runs harnesses and workflows set up for them, adapts rules and settings, reviews outputs.</li>
  <li><b>Builder:</b> sets up and shapes harnesses: creates projects, writes rules and procedures.</li>
  <li><b>Engineer:</b> changes and reviews production code, builds from scratch, merges and approves PRs.</li>
</ul>
<h2>Safety rules</h2>
<ul>
  <li>Patient and personal data (PHI/PII) is never stored in the repository.</li>
  <li>Sensitive AI processing runs only through approved channels (AWS Bedrock).</li>
  <li>Agents never email, message or share files with people outside Zorro.</li>
  <li>Each agent has its own limited Google identity. Secrets are kept in 1Password or the Keychain.</li>
  <li>Agents work only inside their assigned project.</li>
</ul>

<div class="two">
  <div>
    <h2>Strengths</h2>
    <ul>
      <li>One consistent pattern for every project.</li>
      <li>Governance built in from the start.</li>
      <li>Non-engineers can run agent projects.</li>
      <li>Real tools already in use.</li>
    </ul>
  </div>
  <div>
    <h2>Open risks</h2>
    <ul>
      <li>Most of the work depends on one person.</li>
      <li>Setup and tools assume macOS.</li>
      <li>Most apps run locally or from Drive, not hosted.</li>
      <li>AI policies are still drafts; some PHI flows await compliance decisions.</li>
    </ul>
  </div>
</div>

<h2>Current state and next decisions</h2>
<p>Several tools are already in use by sales and operations. The next step is a shared hosting platform ("Deployable") for the most-used apps. Open decisions: which apps to host first, adding a second owner for key projects, and whether Windows support is needed.</p>

<p class="foot">Details for every project (what it is, how it's used, inputs and outputs, credentials, examples, procedures by role): <b>agentlab-guide.md</b>. Questions: Slack <b>#maya-ai-projects</b>.</p>

</body>
</html>
