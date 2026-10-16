# Set up the .NET agents in GitHub Copilot

This guide gets the `dotnet-agents` plugin working on your own machine with GitHub Copilot, in VS Code or Copilot
CLI. It takes about 15 minutes.

> **Status:** the plugin is version 1.0.0 and in its testing phase. If a step behaves differently on your machine,
> tell the plugin team. It helps the live checks.

---

## 1. What you need

| Item | Check |
|---|---|
| GitHub Copilot **Business** licence | Signed in to GitHub in VS Code |
| **VS Code** with the **GitHub Copilot Chat** extension, recent version | Help > About; Extensions view |
| **.NET SDK 10** | `dotnet --version` shows 10.x |
| **Git** | `git --version` |
| Optional: **GitHub Copilot CLI** | `copilot --version` |
| Optional: **GitHub CLI**, signed in (for pull requests) | `gh auth status` |

On Windows, allow long paths in Git once: `git config --global core.longpaths true`.

---

## 2. Get the developer commands (once)

You approve gates and record decisions with the `agents` command in your own terminal. The agents cannot run it.

```
git clone https://github.com/karthikeya-reddy-bommireddy/DotNet-Plugin-Warehouse-Techwave.git
dotnet tool install --global Agents.Cli --add-source DotNet-Plugin-Warehouse-Techwave/cli
agents --help
```

Keep the clone: you need it to update the CLI later.

---

## 3. Install the plugin (choose one way)

### Way A: with Copilot CLI (recommended)

```
copilot plugin marketplace add karthikeya-reddy-bommireddy/DotNet-Plugin-Warehouse-Techwave
copilot plugin install dotnet-agents@techwave-dotnet
copilot plugin list
```

VS Code also picks up the plugins Copilot CLI installed. Restart VS Code afterwards.

### Way B: in VS Code only

1. Open the Command Palette (`Ctrl+Shift+P`) and run **Chat: Install Plugin From Source**.
2. Enter `https://github.com/karthikeya-reddy-bommireddy/DotNet-Plugin-Warehouse-Techwave`.
3. Restart VS Code.

Agent plugins must be on: setting `chat.plugins.enabled` (it is on by default).

### Way C: as project files (Visual Studio, or when plugins are blocked)

Copy the contents of `project-files/copilot/` from the clone into the root of your project: the `.github/` and
`.vscode/` folders. If the project already has a `.vscode/settings.json`, merge its one setting into yours instead of
replacing the file. This way uses the `agents` command from step 2.

Then check the installation:

```
agents verify
```

It compares the installed plugin with its release manifest. "all … file(s) match" means it is good.

---

## 4. VS Code settings

Open **Settings** (`Ctrl+,`) and check these:

| Setting | Value | Why |
|---|---|---|
| `chat.plugins.enabled` | on | Loads the plugin |
| `chat.useHooks` | on | The plugin's safety hooks (scope, secrets, Git rules) run |
| `chat.tools.global.autoApprove` | **off** | You must see and confirm every action |
| Workspace trust | Trust the project folder | Hooks run only in trusted workspaces |

Never use Autopilot or "approve for this workspace" for the engine's tools. Approvals and Git actions rely on you
seeing each step.

---

## 5. Prepare a project (once per project)

1. Open the .NET repository in VS Code. It must be a Git repository with at least one commit.
2. Put the BA package in it, for example `ba.json` (from the BA plugin).
3. In a terminal in the project folder:

   ```
   agents init
   ```

   This creates `.agents/`. Run state stays out of Git; the configuration is committed. The plugin's hooks act only in
   projects that have this folder.

---

## 6. First run

1. Open **Copilot Chat** and switch to **Agent** mode.
2. Pick **dotnet-orchestrator** in the agent list. In Copilot CLI, use `@dotnet-orchestrator`.
3. The first time, VS Code asks to start the MCP server **agents-engine**: allow it. You can see its tools in the
   tools picker.
4. Type:

   ```
   Start a run for the Orders module with ba.json, greenfield.
   ```

   For an existing codebase, say **brownfield** instead.
5. Follow the orchestrator. At each gate it shows the command to run in your terminal, for example
   `agents approve R-0001 gate-1`.

`agents status` shows your runs and what waits for you at any time.

---

## 7. Update and uninstall

```
copilot plugin uninstall dotnet-agents
copilot plugin marketplace update techwave-dotnet
copilot plugin install dotnet-agents@techwave-dotnet

git -C DotNet-Plugin-Warehouse-Techwave pull
dotnet tool update --global Agents.Cli --add-source DotNet-Plugin-Warehouse-Techwave/cli
```

To uninstall: `copilot plugin uninstall dotnet-agents`, `copilot plugin marketplace remove techwave-dotnet` and
`dotnet tool uninstall --global Agents.Cli`. A project keeps its `.agents/` folder; delete it if you no longer need it.

---

## 8. Troubleshooting

| Problem | Try |
|---|---|
| `dotnet-orchestrator` is not in the agent list | Restart VS Code; check `copilot plugin list`; check that `chat.plugins.enabled` is on |
| "agents-engine" does not start | Check `dotnet --version` (needs 10.x); allow the server when VS Code asks; see the Output panel, MCP logs |
| `agents` is not found | Open a new terminal after `dotnet tool install`; check that the .NET tools folder is on your PATH |
| The orchestrator says `action_not_allowed` | Expected: the engine allows only the next step. Ask "what is next?" |
| A hook denies something | Read the message: it names the rule (for example H-05, outside the task's scope) and what to do |
| `git clone` fails with "Filename too long" | `git config --global core.longpaths true` and clone again |
| `agents verify` reports changed files | Reinstall the plugin (step 3) |

For how the agents work and how to run greenfield and brownfield projects, see
[architecture-and-usage.md](architecture-and-usage.md).
