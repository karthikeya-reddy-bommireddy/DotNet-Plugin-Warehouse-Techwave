---
name: decision-request
description: Use when a report has status needs_decision, when .NET technology, versions, libraries or patterns must be chosen, or when escalation needs a developer choice. Prepares 2-3 options from the approved list with pros, cons and one recommendation, batched per gate, and records them with request_decision. Do not use for gate approvals (use gate-package) or for open questions without discrete options.
license: Proprietary
metadata:
  version: 1.1.0
  agent: dotnet-orchestrator
  contract: decision-request/1
disable-model-invocation: true
---
# Decision request

The developer ALWAYS decides technology, versions, scope, design revisions and Git actions.
You prepare the choice; you never make it.

## Steps
1. Collect the decisions from `needs_decision` reports and from escalation.
2. For a technology choice, call `get_approved_options(category)`:
   - `fixed: true`: the company decided. State the option in designs and briefs; raise no card.
   - fewer than 2 `options`: say so and ask whether the approved list should be extended. That is a
     policy decision for a person, not for you.
   - otherwise offer 2 or 3 of those options, never anything else.
   Choices that are not technology use the category `escalation`, `design-revision` or `scope`.
3. For each option give: a short label (and `version` where the developer must pick one), at most 3
   pros, at most 3 cons and the impact on this run (what changes in design, code or tests).
4. Give ONE `recommended` option tied to requirement or NFR IDs in `linked_requirements`, for example
   "recommended because NFR-001 needs ...". Call it "recommendation", never "default" or "chosen".
5. Batch decisions that belong to the same gate into one decision-request/1:
   `schema`, `run_id`, `gate`, and `items` with `id` (DEC-nnn, as the report named it), `category`,
   `question`, `options`, `recommended` and `linked_requirements`.
6. Call `request_decision(run_id, batch)`. If it fails with `invalid_decision`, fix exactly the listed
   `details` and call it once more. On success the run pauses.
7. Show each decision card with the `command` the result gives, and end the turn. A decision counts only
   when `get_run_state` no longer lists it in `pending_decisions`.

## Decision card format
```
DEC-004  Integration-test database (blocks: Gate 1)            linked: REQ-002, NFR-001
  A) <option label> <version>  - pros: ...  cons: ...  impact: ...
  B) <option label> <version>  - pros: ...  cons: ...  impact: ...
  Recommendation: A, because <reason tied to IDs>.
  Decide with: agents decide <run> DEC-004 <option-id>
```

## Never
Offer unapproved options, ask about a fixed category, imply a decision is made, mix approvals into
decisions, or continue before the decision is recorded.
