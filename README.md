# Confirm Deadline

The final report deadline is April 3rd, 2026.

# Project 4 Reverse Engineering Report: Superpowers

* **Project Name:** Superpowers
* **Repository:** https://github.com/obra/superpowers
* **Project Category:** Agent Workflow / Developer Tooling
* **Deadline:** April 3rd, 2026
* **Repository Snapshot Analyzed:** commit `b7a8f76` (`v5.0.7` branch tip at the time of analysis)

## 1. Project Overview and Key Components

### Repository Analysis Summary

Superpowers is not a single application in the usual sense. It is a workflow layer for coding agents: a collection of skills, prompts, plugin manifests, hooks, and platform adapters that try to force a more disciplined software-development process. The top-level [README.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/README.md#L5) describes the intended flow clearly: brainstorm first, turn the approved design into a detailed plan, execute with either subagents or a simpler fallback, review the work, then finish on a branch rather than directly on the main line.

The repository is organized around that workflow:

- [skills/brainstorming/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/brainstorming/SKILL.md) defines the design phase.
- [skills/writing-plans/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/writing-plans/SKILL.md) turns a design into tiny, executable steps.
- [skills/subagent-driven-development/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/subagent-driven-development/SKILL.md) and [skills/executing-plans/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/executing-plans/SKILL.md) define two execution models.
- [skills/requesting-code-review/code-reviewer.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/requesting-code-review/code-reviewer.md) and [agents/code-reviewer.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/agents/code-reviewer.md) define review behavior.
- [.claude-plugin/plugin.json](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/.claude-plugin/plugin.json), [.cursor-plugin/plugin.json](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/.cursor-plugin/plugin.json), [gemini-extension.json](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/gemini-extension.json), [docs/README.codex.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/docs/README.codex.md), and [docs/README.opencode.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/docs/README.opencode.md) show that the project is intentionally multi-platform rather than tied to one agent harness.
- [RELEASE-NOTES.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/RELEASE-NOTES.md#L18) is unusually important in this repo because the design rationale for several workflow changes is documented there explicitly, including the replacement of the old subagent review loop.

In short, Superpowers is best understood as an algorithm for organizing agent behavior, not just as a library of instructions.

## 2. Deep Reasoning Questions & Analysis

### Q1. Superpowers replaced its subagent review loop (which doubled execution time by ~25 minutes) with inline checklists that catch 3-5 bugs per session in 30 seconds. What data structure change underlies this optimization, and why couldn't a smarter review algorithm achieve the same result?

The underlying change is a shift from a recursive review graph to a fixed inline checklist attached to document construction.

- Evidence from [RELEASE-NOTES.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/RELEASE-NOTES.md#L18):

> “The subagent review loop ... doubled execution time (~25 min overhead) without measurably improving plan quality.”

> “Self-review catches 3-5 real bugs per run in ~30s instead of ~25 min...”

Before the change, the workflow for specs and plans had a separate reviewer pass: create the document, dispatch another agent, wait for feedback, revise, and possibly repeat. The release notes spell that out almost directly: Superpowers used a “subagent review loop” for plans and specs, and that loop imposed about 25 minutes of overhead without measurable quality improvement ([RELEASE-NOTES.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/RELEASE-NOTES.md#L18)). In `v5.0.0`, the earlier design was even more explicit: spec and plan review were looped subagent workflows with repeated review until approval ([RELEASE-NOTES.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/RELEASE-NOTES.md#L239)).

The replacement turns that open-ended loop into a compact in-place structure:

- Brainstorming now ends with a four-item self-review checklist: placeholder scan, internal consistency, scope check, and ambiguity check ([skills/brainstorming/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/brainstorming/SKILL.md#L116)).
- Writing-plans now ends with a three-item self-review checklist: spec coverage, placeholder scan, and type consistency ([skills/writing-plans/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/writing-plans/SKILL.md#L122)).

That matters because the new workflow stores review state as a bounded list of invariants inside the authoring pass itself, instead of storing review as another node in the control-flow graph. The difference is structural:

- Old shape: draft -> external reviewer -> feedback -> revise -> maybe repeat.
- New shape: draft -> run checklist locally -> patch inline -> done.

In complexity terms, the inline checklist reduces the process from something closer to `O(n * m)` to something much closer to `O(n)`. Here, `n` is the size of the spec or plan and `m` is the number of extra review passes, reviewer-agent invocations, or revision cycles applied to that same document. Under the old loop, the document could be processed repeatedly across multiple review rounds, so the cost scaled with both document size and the number of review iterations. With the inline checklist, the review criteria are fixed and embedded in the same authoring pass, so the document is checked once against a bounded set of invariants. That is why the checklist does not just save time; it reduces the control-flow complexity of the workflow itself.

A smarter review algorithm would not remove the real cost because the expensive part was not just bad review quality. It was the architecture of the loop itself: another dispatch, another context load, another full pass over the document, another synchronization point, and possibly another cycle. The release notes make this point indirectly but strongly: regression testing across five versions and five trials showed “identical quality scores regardless of whether the review loop ran” ([RELEASE-NOTES.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/RELEASE-NOTES.md#L20)). That means the bottleneck was loop overhead, not reviewer intelligence.

**References**

- [RELEASE-NOTES.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/RELEASE-NOTES.md#L18)
- [RELEASE-NOTES.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/RELEASE-NOTES.md#L239)
- [skills/brainstorming/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/brainstorming/SKILL.md#L116)
- [skills/writing-plans/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/writing-plans/SKILL.md#L122)

### Q2. Superpowers breaks work into 2-5 minute task increments in the planning phase. Why is this duration significant from a complexity perspective, and what happens if tasks are either too granular or too coarse?

The 2-5 minute window is a practical upper bound on hidden complexity.

- Evidence from [skills/writing-plans/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/writing-plans/SKILL.md#L36):

> “Each step is one action (2-5 minutes)”

The planning skill is very explicit: “Each step is one action (2-5 minutes)” and then it gives canonical examples such as “Write the failing test,” “Run it to make sure it fails,” “Implement the minimal code,” and “Commit” ([skills/writing-plans/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/writing-plans/SKILL.md#L36)). That is not just project-management advice. It is a constraint on how much reasoning is allowed to hide inside any one step.

From a complexity point of view, this keeps each planning node small enough that:

- the goal is singular,
- the dependency set is visible,
- the verification step is obvious,
- and failure can be attributed cleanly.

If steps get too granular, the graph explodes in the other direction. The agent spends more time on state transitions, bookkeeping, and coordination than on actual work. You get a lot of tiny nodes with almost no semantic distance between them, and the orchestration overhead starts to dominate.

If steps get too coarse, the step stops being a step and becomes a hidden subproject. Then the agent is effectively improvising an internal plan during execution. That defeats the point of explicit planning and reintroduces branching, ambiguity, and untracked dependencies. The writing-plans skill is designed to avoid exactly that by forcing exact file paths, exact commands, and exact expected outcomes ([skills/writing-plans/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/writing-plans/SKILL.md#L63)).

So the 2-5 minute rule is significant because it is the scale at which a step is still atomic for an agent: small enough to reason about directly, but large enough that the orchestration cost does not swamp the work itself.

**References**

- [skills/writing-plans/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/writing-plans/SKILL.md#L36)
- [skills/writing-plans/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/writing-plans/SKILL.md#L63)
- [README.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/README.md#L114)

### Q3. The brainstorming skill requires breaking large systems into “smaller units that each have one clear purpose, communicate through well-defined interfaces, and can be understood and tested independently.” How does this constraint reduce the algorithmic complexity of implementation planning?

It reduces planning complexity by turning one highly coupled problem into several locally solvable ones.

- Evidence from [skills/brainstorming/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/brainstorming/SKILL.md#L94):

> “Break the system into smaller units that each have one clear purpose, communicate through well-defined interfaces, and can be understood and tested independently”

The brainstorming skill says this outright in its “Design for isolation and clarity” section: the system should be broken into units with one clear purpose, well-defined interfaces, and independent comprehensibility and testability ([skills/brainstorming/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/brainstorming/SKILL.md#L94)). The writing-plans skill reinforces the same idea at the file-planning layer, insisting on clear boundaries, one responsibility per file, and decomposition before tasks are written ([skills/writing-plans/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/writing-plans/SKILL.md#L25)).

This reduces algorithmic complexity in three ways:

First, it reduces the number of interactions that need to be considered simultaneously. If everything talks to everything else implicitly, the planner has to reason about a dense web of dependencies. If communication happens only through explicit interfaces, most of the search space collapses to interface contracts and local internals.

Second, it narrows the context window needed for each decision. The skill itself says reasoning is better when the code can be “held in context at once” and edits are more reliable when files are focused ([skills/writing-plans/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/writing-plans/SKILL.md#L29); [skills/brainstorming/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/brainstorming/SKILL.md#L99)).

Third, it makes testing compositional. When a unit can be understood and tested independently, the planner can verify one component without simulating the whole system. That makes the implementation plan more like a sequence of controlled local transformations than one giant coupled search problem.

In other words, the rule is doing more than improving readability. It is explicitly lowering the branching factor of the planning problem.

**References**

- [skills/brainstorming/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/brainstorming/SKILL.md#L94)
- [skills/writing-plans/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/writing-plans/SKILL.md#L25)
- [RELEASE-NOTES.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/RELEASE-NOTES.md#L251)

### Q4. Code review in Superpowers is strictly limited to “125 characters” for source code quotes in feedback. Why is this character limit not arbitrary, and what evaluation problem does it solve?

 The current review template requires every issue to include a `File:line reference`, `What's wrong`, `Why it matters`, and `How to fix`, and its critical rules repeat the same expectation: “Be specific (file:line, not vague)” and “Explain WHY issues matter” ([skills/requesting-code-review/code-reviewer.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/requesting-code-review/code-reviewer.md#L94)). The README describes the same workflow at a higher level: code review happens between tasks, reports issues by severity, and blocks progress on critical problems ([README.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/README.md#L120)).

In that context, a 125-character quote cap is not arbitrary. It is a calibration device. It forces the reviewer to include only the minimum code fragment needed to anchor the issue, then spend the rest of the feedback budget on diagnosis. The review is therefore evaluated on whether it identified the defect correctly, localized it correctly, and explained its impact correctly, not on whether it pasted a long block of source.

That solves a specific evaluation problem: long code quotes can make shallow reviews look strong. A reviewer can copy a large chunk of the diff, point vaguely at it, and appear detailed without actually isolating the bug. Once quote length is tightly capped, that shortcut stops working. The reviewer has to compress the evidence and supply real analysis. In practical terms, the cap separates bug finding from code reproduction.

The limit also improves comparability across reviews. If one reviewer pastes three lines and another pastes thirty, the difference in apparent depth may have more to do with quoting style than review quality. A fixed small cap normalizes that variable and makes reviews easier to judge on substance: accuracy of the issue, severity calibration, and clarity of the explanation.

**References**

- [skills/requesting-code-review/code-reviewer.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/requesting-code-review/code-reviewer.md#L79)
- [skills/requesting-code-review/code-reviewer.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/requesting-code-review/code-reviewer.md#L94)
- [agents/code-reviewer.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/agents/code-reviewer.md)
- [README.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/README.md#L120)


### Q5. Superpowers integrates across Claude Code, Cursor, Codex, OpenCode, and Gemini CLI through platform-specific configuration files (`.claude-plugin/`, `.cursor-plugin/`, etc.) rather than a single abstraction. What platform-specific concerns make a unified abstraction impractical?

A single abstraction would be too leaky because the platforms differ at the level of plugin model, startup injection, tool vocabulary, subagent support, and even JSON field names.

- Evidence from [README.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/README.md#L27):

> “Installation differs by platform.”

The repository shows this everywhere:

- Claude Code has a basic plugin manifest in [.claude-plugin/plugin.json](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/.claude-plugin/plugin.json#L1).
- Cursor needs a different manifest with explicit `skills`, `agents`, `commands`, and `hooks` fields in [.cursor-plugin/plugin.json](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/.cursor-plugin/plugin.json#L1).
- Cursor also expects a separate hook schema file with `sessionStart` in camelCase in [hooks/hooks-cursor.json](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/hooks/hooks-cursor.json#L1).
- Gemini uses [gemini-extension.json](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/gemini-extension.json#L1) plus [GEMINI.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/GEMINI.md#L1) imports rather than a plugin manifest with hooks.
- Codex does not use the same plugin path at all; it relies on native skill discovery through a symlink under `~/.agents/skills/`, and subagent support must be enabled with `multi_agent = true` in [docs/README.codex.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/docs/README.codex.md#L27).
- OpenCode is plugin-driven again, but through `opencode.json`, a `config` hook, and `experimental.chat.system.transform` in [docs/README.opencode.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/docs/README.opencode.md#L91).

The hook code makes the incompatibility even more concrete. The `session-start` script has to emit different JSON keys depending on the runtime:

- Cursor expects `additional_context`,
- Claude Code expects `hookSpecificOutput.additionalContext`,
- Copilot/other SDK-style harnesses expect `additionalContext`

and the script explicitly warns that Claude will double-read certain fields if both forms are emitted ([hooks/session-start](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/hooks/session-start#L37)).

Tool mapping is another hard incompatibility:

- Codex maps `Task` to `spawn_agent`, but it has no named agent registry, so the project needs a workaround that reads prompt files and wraps them manually ([skills/using-superpowers/references/codex-tools.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/using-superpowers/references/codex-tools.md#L27)).
- Gemini has no subagent equivalent at all, so subagent-heavy skills must fall back to `executing-plans` ([skills/using-superpowers/references/gemini-tools.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/using-superpowers/references/gemini-tools.md#L17)).
- OpenCode maps Claude-style tools onto its own native `skill` tool and `@mention` subagent system ([docs/README.opencode.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/docs/README.opencode.md#L98)).

So a unified abstraction is impractical because the differences are not just naming differences. They are differences in execution model, capability surface, lifecycle hooks, and agent orchestration semantics.

**References**

- [.claude-plugin/plugin.json](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/.claude-plugin/plugin.json#L1)
- [.cursor-plugin/plugin.json](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/.cursor-plugin/plugin.json#L1)
- [hooks/hooks-cursor.json](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/hooks/hooks-cursor.json#L1)
- [hooks/session-start](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/hooks/session-start#L37)
- [gemini-extension.json](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/gemini-extension.json#L1)
- [GEMINI.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/GEMINI.md#L1)
- [docs/README.codex.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/docs/README.codex.md#L27)
- [docs/README.opencode.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/docs/README.opencode.md#L7)
- [docs/README.opencode.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/docs/README.opencode.md#L91)
- [skills/using-superpowers/references/codex-tools.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/using-superpowers/references/codex-tools.md#L27)
- [skills/using-superpowers/references/gemini-tools.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/using-superpowers/references/gemini-tools.md#L17)

### Q6. The planning skill breaks work into “numbered planning steps with clear implementation details” that can be executed by separate subagents. Why is this explicit numbering and step separation crucial rather than allowing subagents to improvise their own breakdown?

Because numbered steps are the shared control structure that lets the controller, the implementer, and the reviewers all agree on what “the work” actually is.

- Evidence from [skills/writing-plans/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/writing-plans/SKILL.md#L63):

```markdown
### Task N: [Component Name]
- [ ] Step 1 ...
- [ ] Step 2 ...
```

The planning template is rigid on purpose. It requires `Task N`, then explicit files, then a checklist of numbered steps with concrete code, commands, and expected results ([skills/writing-plans/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/writing-plans/SKILL.md#L63)). The subagent-driven execution model assumes that structure exists beforehand. It says the controller reads the plan, extracts all tasks, creates tracking, dispatches one implementer subagent per task, then runs spec review and code-quality review on that same task boundary ([skills/subagent-driven-development/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/subagent-driven-development/SKILL.md#L40)).

If subagents were allowed to invent their own decomposition on the fly, several things would break:

- There would be no stable unit for progress tracking.
- Reviews would be comparing implementation against a moving target.
- Rework would be hard to assign back to a particular plan step.
- Different subagents could split the same task differently, which would make controller-level coordination much harder.

The numbered structure is therefore not formatting ceremony. It is what makes task identity stable across dispatch, implementation, verification, and re-review.

The repo hints at this more broadly too. The plan is supposed to be detailed enough for “an enthusiastic junior engineer with poor taste” to execute safely ([README.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/README.md#L11)). That only works if the breakdown has already been fixed upstream.

**References**

- [skills/writing-plans/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/writing-plans/SKILL.md#L63)
- [skills/subagent-driven-development/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/subagent-driven-development/SKILL.md#L40)
- [skills/subagent-driven-development/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/subagent-driven-development/SKILL.md#L128)
- [README.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/README.md#L11)

### Q7. The brainstorming skill offers to present designs “in digestible sections for approval” rather than a complete specification first. Why is incremental approval better than requiring full spec review upfront?

Incremental approval is better because it reduces review load and catches structural misunderstandings before they infect the rest of the design.

- Evidence from [README.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/README.md#L7):

> “it shows it to you in chunks short enough to actually read and digest”

The repo is explicit about this. The README says the spec is shown in chunks “short enough to actually read and digest” ([README.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/README.md#L7)). The brainstorming skill says design should be presented in sections, scaled to complexity, with approval requested after each section ([skills/brainstorming/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/brainstorming/SKILL.md#L28); [skills/brainstorming/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/brainstorming/SKILL.md#L86); [skills/brainstorming/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/brainstorming/SKILL.md#L144)).

That is a design choice with real consequences:

- A long complete spec makes the reviewer hold the entire design in working memory.
- If an early assumption is wrong, the rest of the document may already be contaminated by it.
- Fixing that mistake late is much more expensive than correcting it after one section.

Incremental approval turns spec review into a sequence of local validations instead of one giant acceptance event. That lowers cognitive load and lowers the cost of correction. The structure in `brainstorming/SKILL.md` shows this clearly: “present design sections” is followed by “user approves design?” and loops back for revision if the answer is no ([skills/brainstorming/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/brainstorming/SKILL.md#L43)).

So this is not just about readability. It is a way of keeping design convergence cheap.

**References**

- [README.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/README.md#L7)
- [skills/brainstorming/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/brainstorming/SKILL.md#L28)
- [skills/brainstorming/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/brainstorming/SKILL.md#L43)
- [skills/brainstorming/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/brainstorming/SKILL.md#L86)
- [skills/brainstorming/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/brainstorming/SKILL.md#L144)

### Q8. Superpowers stores design specifications in timestamped files (`docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`). Why is the timestamp part of the filename critical, and what problem does it prevent?

The timestamp gives each design artifact a stable identity in time, which prevents overwrite, ambiguity, and broken linkage between designs and plans.

The convention is stated directly in the skills:

- Brainstorming writes specs to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` ( [skills/brainstorming/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/brainstorming/SKILL.md#L111)).
- Writing-plans writes plans to `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md` ([skills/writing-plans/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/writing-plans/SKILL.md#L18)).
- The release notes also call out this path restructuring as a major workflow change ([RELEASE-NOTES.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/RELEASE-NOTES.md#L207)).

The actual repository contents show the convention in use:

- `docs/superpowers/specs/2026-01-22-document-review-system-design.md`
- `docs/superpowers/specs/2026-02-19-visual-brainstorming-refactor-design.md`
- `docs/superpowers/specs/2026-03-11-zero-dep-brainstorm-server-design.md`
- `docs/superpowers/specs/2026-03-23-codex-app-compatibility-design.md`

Without the timestamp, repeated work on the same topic would create collisions. A generic filename like `codex-app-compatibility-design.md` would not tell you whether you were reading the first design, the revised one, or the one that actually generated the plan you are implementing. The timestamp makes the document set behave more like an append-only design log than a set of mutable placeholders.

That matters in Superpowers because the spec is not a dead document. It is the input to planning, review, and execution. If its identity is ambiguous, the whole downstream chain becomes ambiguous too.

**References**

- [skills/brainstorming/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/brainstorming/SKILL.md#L29)
- [skills/brainstorming/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/brainstorming/SKILL.md#L111)
- [skills/writing-plans/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/writing-plans/SKILL.md#L18)
- [RELEASE-NOTES.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/RELEASE-NOTES.md#L207)
- [docs/superpowers/specs](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/docs/superpowers/specs)
- [docs/superpowers/plans](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/docs/superpowers/plans)

### Q9. The executing-plans skill explicitly forbids implementation on main/master branch without user consent. Why is this a safety constraint in the skill itself rather than just developer discipline?

Because Superpowers is built on the assumption that process constraints must be encoded into the workflow, not left to good intentions.

The executing-plans skill puts this directly in its “Remember” section: “Never start implementation on main/master branch without explicit user consent” ([skills/executing-plans/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/executing-plans/SKILL.md#L57)). The subagent-driven-development skill repeats the same rule under its red flags ([skills/subagent-driven-development/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/subagent-driven-development/SKILL.md#L234)). The README also frames branch-based isolation as a mandatory part of the pipeline through the `using-git-worktrees` step ([README.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/README.md#L112)).

That placement matters. The rule is inside the skill because the skill is the executable policy. Once the agent enters the workflow, the skill is what constrains its actions. If the rule were only a team norm, it could be forgotten, skipped under time pressure, or bypassed by an overconfident agent run.

This repository keeps making the same design move: important behavior is pushed into mandatory workflow instructions. The README says these are “mandatory workflows, not suggestions” ([README.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/README.md#L124)). So branch protection belongs in the skill for the same reason test-first behavior or explicit verification belongs in a skill: it is a guardrail that should survive time pressure and automation.

**References**

- [skills/executing-plans/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/executing-plans/SKILL.md#L57)
- [skills/subagent-driven-development/SKILL.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/skills/subagent-driven-development/SKILL.md#L234)
- [README.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/README.md#L112)

### Q10. Superpowers’ version history (v1 through v5) shows major version changes, including the replacement of a costly review loop with checklists. Why is accepting that regressions (some bugs slip through) a necessary trade-off in this direction?

Because a workflow that is too expensive to run consistently is worse than a slightly weaker workflow that actually gets used.

The release notes are blunt about the trade-off. In `v5.0.6`, the subagent review loop was removed because it added about 25 minutes of overhead without measurable gains in plan quality, while the inline self-review still caught 3-5 real bugs in around 30 seconds with comparable defect rates ([RELEASE-NOTES.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/RELEASE-NOTES.md#L18)). That is almost a textbook case of choosing lower latency and higher throughput over maximal gate strength.

The longer history also shows that the project has repeatedly changed structure when the previous architecture became too heavy or too brittle:

- `v5.0.0` restructured spec and plan storage and changed execution defaults ([RELEASE-NOTES.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/RELEASE-NOTES.md#L207)).
- Earlier `v5.0.0` notes added formal document review loops as a new system ([RELEASE-NOTES.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/RELEASE-NOTES.md#L239)).
- `v5.0.6` then simplified those loops after observing the actual cost/benefit ratio ([RELEASE-NOTES.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/RELEASE-NOTES.md#L18)).
- The tag history shows sustained architectural change across v1, v2, v3, v4, and v5 rather than a static design.

So the project’s direction is not “never let a bug through.” It is closer to “design the cheapest process that still catches the bugs worth catching.” If every extra layer of review adds huge waiting costs, then even a theoretically stronger workflow can become net worse because:

- fewer cycles get completed,
- feedback arrives too late,
- users are tempted to skip the process,
- and total throughput falls.

Accepting some regressions is therefore not carelessness. It is the cost of moving toward a workflow with a better end-to-end operating point.

**References**

- [RELEASE-NOTES.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/RELEASE-NOTES.md#L18)
- [RELEASE-NOTES.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/RELEASE-NOTES.md#L207)
- [RELEASE-NOTES.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/RELEASE-NOTES.md#L239)
- [README.md](https://github.com/obra/superpowers/blob/b7a8f76985f1e93e75dd2f2a3b424dc731bd9d37/README.md#L152)
- `git tag --sort=v:refname`

## 3. Findings and Conclusion

Superpowers is best understood as a workflow optimizer for agentic software development. Most of its important design choices are not about code generation itself. They are about keeping the control structure of development manageable: shrinking task size, separating concerns, introducing review at the right granularity, and adapting the same high-level process to very different host environments.

Three patterns came through consistently in the repository:

First, the project treats complexity as something to be managed structurally rather than heroically. Small steps, isolated units, explicit file boundaries, and staged approval all exist to keep reasoning local instead of global.

Second, the project is willing to trade theoretical thoroughness for practical throughput when the empirical data supports it. The shift from subagent review loops to inline self-review is the clearest example: Superpowers gave up a heavier control mechanism because the measured returns did not justify the cost.

Third, the repository is deeply platform-aware. Claude Code, Cursor, Codex, OpenCode, and Gemini are not treated as skins over one universal runtime. They are treated as different harnesses with different plugin models, hook systems, tool vocabularies, and orchestration constraints. That is why the repository contains multiple manifests, multiple install paths, multiple hook formats, and multiple tool-mapping documents.
