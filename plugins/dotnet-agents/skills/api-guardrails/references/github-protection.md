# GitHub protection: CODEOWNERS, branch rules, secret scanning

> **Placeholders:** `<owners>` and folder paths are illustrations. Code owners must be real GitHub users or teams with write access, given by the user. Never guess a handle.

## CODEOWNERS template (`.github/CODEOWNERS`)
Guardrail files enforce security, and Copilot instruction files steer every developer's Copilot, so changes to either need review by the security owners.
```
# Copilot instruction files. In this project the agent is installed locally and never committed,
# so any pull request that adds these files needs security review.
/.github/agents/                   <owners>
/.agents/plugin/skills/                   <owners>
/.github/prompts/                  <owners>
/.github/instructions/             <owners>
/.github/copilot-instructions.md   <owners>
/.agents/api-layer/          <owners>
/.github/CODEOWNERS                <owners>

# Other folders Copilot can read instructions or skills from
/.claude/                          <owners>
/.agents/                          <owners>
/AGENTS.md                         <owners>

# Guardrails
/Directory.Build.props             <owners>
/.editorconfig                     <owners>
/eng/security/                     <owners>
/eng/pipelines/                    <owners>
/<guardrail tests folder>/         <owners>
```
- In CODEOWNERS the last matching line wins, so keep these lines below any broad rules.
- CODEOWNERS blocks merges only when the branch rule "Require review from Code Owners" is on.

## Branch-rule checklist (for a repo admin)
Apply this to the default branch and release branches, through branch protection or rulesets:
- [ ] Require a pull request before merging (the number of approvals is a team decision)
- [ ] Require review from Code Owners
- [ ] Dismiss stale approvals when new commits are pushed
- [ ] Require status checks to pass, and select the Azure Pipelines security-gates check (it appears after the pipeline has run once on a pull request)
- [ ] Block force pushes and branch deletion
- [ ] Decide who, if anyone, may bypass these rules (team decision)

The agent cannot apply these settings. Report them as handed over, never as done.

## Secret scanning (team decision; ask)
1. **GitHub secret scanning with push protection**: blocks pushes that contain secrets, and scans history. For private repositories, availability depends on the GitHub plan.
2. **A scanner step in the security stage** (open-source or commercial), pinned to a version.

If a secret is found, rotate it first; deleting it from the code is not enough, because it stays in git history.

## Optional (team decision)
Dependabot alerts and security updates for NuGet packages, alongside the NuGet audit in the build.
