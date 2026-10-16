# TechWave .NET agents

A plugin marketplace for **Claude Code** and **GitHub Copilot** (Copilot CLI, and VS Code through its agent plugins)
with one plugin, `dotnet-agents`: an orchestrator and eight specialist agents that take a BA package through .NET
adaptation, module design, a build loop (code, review, test, fix), security and documentation to three approval gates
and your Git decision. A deterministic engine (an MCP server inside the plugin) owns the rules, gates and checks; you
make every decision and approval.

## Guides

- [Set up the agents in GitHub Copilot](docs/setup-github-copilot.md): a step-by-step local setup.
- [Architecture, flow and usage](docs/architecture-and-usage.md): how the agents work, and how to run greenfield and
  brownfield projects.
- `project-files/`: the same agents as files to copy into a project, for hosts without plugin support (Visual Studio).

## Before you start

- **.NET SDK** (the version this release was built with, see `plugins/dotnet-agents/release-manifest.json`) and **Git**.
- **Claude Code** or **GitHub Copilot CLI** (for VS Code: GitHub Copilot Chat with agent plugins enabled).
- Business plans only (GitHub Copilot Business, Claude Team).

## Install (once per machine)

**1. Add the marketplace**

```
claude plugin marketplace add karthikeya-reddy-bommireddy/DotNet-Plugin-Warehouse-Techwave
copilot plugin marketplace add karthikeya-reddy-bommireddy/DotNet-Plugin-Warehouse-Techwave
```

**2. Install the plugin**

```
claude plugin install dotnet-agents@techwave-dotnet
copilot plugin install dotnet-agents@techwave-dotnet
```

VS Code: run **Chat: Install Plugin From Source** and enter `https://github.com/karthikeya-reddy-bommireddy/DotNet-Plugin-Warehouse-Techwave`, or install it
with Copilot CLI (VS Code finds plugins Copilot CLI installed).

**3. Install the developer commands** (`agents approve`, `agents decide`, `agents rerun`, ...). Only you run these, in
your own terminal; the agents cannot. Download this repository (or clone it), then:

```
dotnet tool install --global Agents.Cli --add-source <this-repository>/cli
```

**4. Restart** the AI tool. Check with `claude plugin list` or `copilot plugin list`, and `agents verify` (compares the
installed files with the release manifest).

**Optional, Claude Code:** `agents init --permissions` adds the plugin's Git deny and ask rules to your user settings
(plugins cannot ship them); it shows the rules and asks first.

## Use

Open a .NET repository (a Git repository with a commit) and tag the orchestrator:

- Claude Code: `@agent-dotnet-agents:dotnet-orchestrator start a run for the Orders module with ba.json`
  (or `claude --agent dotnet-agents:dotnet-orchestrator`)
- Copilot: pick **dotnet-orchestrator** in the agent list, or `@dotnet-orchestrator` in Copilot CLI.

The first run creates `.agents/` in the project (or run `agents init`); the plugin's hooks act only in projects that
have it. The orchestrator tells you at each gate which command to run (`agents approve <run> gate-1` and so on).
`agents status` lists your runs. Other developer commands: `agents rerun <run>` (the BA package changed),
`agents rollback <run>`, `agents lesson list | approve | reject` (lessons from run retrospectives) and
`agents metrics` (what your runs measure against the team's targets).

**Keep the AI tool in a mode that asks before it acts.** Claude Code: Manual mode (not auto mode, `acceptEdits` or
`bypassPermissions`). VS Code: no global auto-approval or Autopilot. Copilot CLI: no `--allow-all-tools` or `--yolo`.

## Update

```
claude plugin marketplace update techwave-dotnet
claude plugin update dotnet-agents@techwave-dotnet

copilot plugin uninstall dotnet-agents
copilot plugin marketplace update techwave-dotnet
copilot plugin install dotnet-agents@techwave-dotnet

dotnet tool update --global Agents.Cli --add-source <this-repository>/cli
```

To go back to an earlier release, add the marketplace at its tag, for example
`claude plugin marketplace add karthikeya-reddy-bommireddy/DotNet-Plugin-Warehouse-Techwave@v1.0.0`, and install again.

## Uninstall

```
claude plugin uninstall dotnet-agents@techwave-dotnet
claude plugin marketplace remove techwave-dotnet
copilot plugin uninstall dotnet-agents
copilot plugin marketplace remove techwave-dotnet
dotnet tool uninstall --global Agents.Cli
```

A project keeps its `.agents/` folder (run state is git-ignored there); delete it if you no longer need the records.
