---
name: code-graph
description: Find code, callers and change impact in this .NET repo through the local Graphify code graph (graphify query, explain, path, update) instead of opening many files. Use at the start of any task that must locate classes, methods, endpoints or dependencies, and whenever a plan needs its impact list.
license: Proprietary
metadata:
  version: 1.7.1
  owner: api-layer-team
  agent: dotnet-coder
  contract: code-graph/1
  source: api-layer-engineer v1.7.1 (body unchanged; frontmatter added by the warehouse)
disable-model-invocation: true
---

# Code graph (Graphify)

> **Placeholders:** example names and numbers are illustrations. Use project-facts or ask; never copy them into code.

Team-adopted (see project-facts): latest Graphify version, code only, command line only, a local graph that is never committed.

## Rules (most important first)
1. **Graph answers are hints, not facts.** Confirm every file, symbol and relationship in the real code before you plan or change anything. The graph can be stale or miss links (reflection, convention-based DI registration, generated code).
2. **Never install or reconfigure Graphify.** Do not run `graphify install`, `graphify vscode install`, `graphify hook install`, package installs (`pip`, `uv`, `pipx`) or the HTTP server (`--transport http`): they write shared files or expose the graph. Never set an LLM key for Graphify; the graph is code only.
3. **Never commit or paste graph files.** `graphify-out/` and `.graphifyignore` stay local (listed in `.git/info/exclude`). Quote only the small results you need.
4. **Graph missing or a command fails:** say so, point the user to "Code graph setup" in `.agents/api-layer/README.md`, continue with normal search, and write "graph unavailable" under Facts used.
5. **Works with the latest Graphify.** If a command fails or its options changed, check the built-in help (`graphify --help` or `graphify <command> --help`) and use the current equivalent. If the error points to an old or incompatible graph, ask the user to rebuild with `graphify extract .`. If nothing works, use normal search. Never install or upgrade Graphify yourself.

## How to use it
1. **Freshness:** if `graphify-out/graph.json` exists, run `graphify update .` once per task; it refreshes only the local graph, never project files. In read-only or no-approval mode, skip the update and query the existing graph.
2. **Find:** `graphify query "<question>"`, for example `graphify query "which controllers use IOrderRepository"`.
3. **One concept:** `graphify explain "<Name>"` shows what the graph knows about that node and its connections.
4. **Trace:** `graphify path "<A>" "<B>"`, for example from a controller to the DbContext, to see which layers a change crosses.
5. **New area:** skim `graphify-out/GRAPH_REPORT.md` for central files and clusters.
6. **Then** open only the files and line ranges the graph pointed to.

## Impact list for the plan
For each symbol the plan changes (entity, DTO, port, repository method, endpoint), run `graphify explain` and list its callers and dependents, confirmed in code. When a public contract changes (DTO shape, route, port signature), list every affected caller and its tests.

## Useful questions
"where is <Entity> persisted" · "what calls <Method>" · "which endpoints use <Policy>" · "what depends on <Interface>" · "which tests cover <Class>"
