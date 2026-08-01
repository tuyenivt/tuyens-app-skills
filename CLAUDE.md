# CLAUDE.md

Guidance for Claude Code working in this repository.

## Overview

A **Claude Code plugin marketplace repository** for mobile client development - agent skills and agents for Claude Code and Codex. Contains only Markdown skill definitions (`.md`); no application code, no build scripts, no test runners.

## Repository Structure

```
plugins/
  flutter/       # Flutter / Dart 3.x - client plugin (mobile primary, desktop secondary, web tertiary)
  unity/         # Unity 6.3 LTS / C# - client plugin, 2D games (mobile primary, desktop secondary, WebGL tertiary)
```

Each plugin folder has a `README.md`. Each skill lives in its own directory as `SKILL.md`. Agent files are plain Markdown in `plugins/<stack>/agents/`.

**Plugin independence rule: every plugin is self-contained and depends on nothing.** Exactly one plugin is installed per project - `flutter` and `unity` are never installed together - so neither may reference skills, agents, or slash commands from the other. Cross-references like that never resolve at install time and must not be authored. When both plugins need the same behaviour, each carries its own copy (see Skill Placement); duplication is the accepted cost of standalone installs.

**Both client plugins are authored against the client domain** rather than adapted from a backend plugin - transactions, connection pools, and server middleware do not map to a client. Neither reviews API contract design: a client consumes API contracts rather than designing them. The concern that does reach clients - an installed old app version must survive a server contract change - is checked inside the umbrella review and `task-<stack>-implement`. Accessibility is handled in `task-<stack>-implement` and checked at baseline depth in the umbrella's Phase E, alongside adaptivity and localization.

**Review lenses are perf and security only.** The umbrella review auto-escalates into `task-<stack>-review-perf` and `task-<stack>-review-security`; there is no observability or reliability lens. The surviving scope enum is `core-only`, `+perf`, `+sec`, `full`.

**Reviews read the working tree.** Uncommitted changes are the subject, not an obstacle. There is no branch-versus-base comparison, no PR requirement, and no incremental re-review checkpoint - `review-precondition-check` resolves the change set (`working-tree`, `staged-only`, or a `last-commit` fallback on a clean tree) and the workflow diffs against it.

`unity` differs from `flutter` in being engine-and-asset-centric: scenes, prefabs, and ScriptableObjects are hand-authored review surface, not generated output. Its central technical opinion is the **engine-free rules core** - game rules are plain C# with no `UnityEngine` dependency, enforced by an assembly definition - which is what makes a wide genre range testable without Play mode. It targets Unity 6.3 LTS (`6000.3.x`) as a hard floor and UI Toolkit only; uGUI and pre-6.3 engines are out of scope by design.

## Skill File Format

```yaml
---
name: skill-name
description: Short description shown in skill picker
metadata:
  category: backend
  tags: [tag1, tag2]
  type: workflow # only for task-* workflow skills
user-invocable: true # false = atomic skill, hidden from slash menu
---
```

- **Workflow skills** (`task-*`, `user-invocable: true`): user-facing slash commands that orchestrate atomic skills end-to-end.
- **Atomic skills** (`user-invocable: false`): focused single-concern patterns, composed via `Use skill: <name>` directives in Markdown bodies.

**Naming.** Workflow skills are prefixed `task-` (e.g., `task-flutter-implement`). Atomic skills use `<framework>-<concern>` (e.g., `flutter-riverpod-patterns`).

**Project detection.** Skills that need to confirm the project type read its marker file directly: `pubspec.yaml` for Flutter, `ProjectSettings/ProjectVersion.txt` and `Packages/manifest.json` for Unity. There is no shared detection skill - each plugin targets exactly one ecosystem.

## Skill Placement

Every skill lives in the plugin that uses it. There is no shared plugin, so a skill both plugins need is **duplicated** into each, under the same name.

When editing a duplicated skill (currently `behavioral-principles` and `review-precondition-check`), apply the same change to both copies. They are expected to stay identical; divergence is a defect unless it is engine-specific by design.

Skills are resolved by name within the installed plugin, so a `Use skill: <name>` reference only resolves when the target ships in that same plugin. Before adding a reference, confirm the target directory exists under the *same* `plugins/<stack>/skills/` tree.

## Environment

- **Shell:** Git Bash on Windows. Use Unix commands (`mv`, `cp`, `mkdir -p`, forward slashes). No PowerShell or CMD.
- **Git: read-only.** Run only read operations (`git log`, `git diff`, `git status`, `git blame`). Never run state-changing git commands - the user manages all commits and branches.

## Writing Conventions

- Use `-` (hyphen-minus). Never `—` or `–` (em/en dash) in any Markdown file.

## Behavioral Principles

How Claude reasons and acts in this repo, in addition to the technical rules above.

- **Think before acting.** State assumptions before editing. If a request has multiple interpretations, present them. Read a skill before editing it; confirm referenced skills exist; count before claiming a number.
- **Minimum change, surgical scope.** Make the smallest edit that satisfies the request. Don't reformat untouched files, bump unrelated versions, or improve adjacent skills. Match existing conventions even if you'd do them differently.
- **Surface confusion, don't paper over it.** When skills contradict, frontmatter is missing, a `Use skill:` target doesn't exist, or versions disagree across `plugin.json`/`marketplace.json` - stop and name it. Don't silently pick a side.
- **Present tradeoffs.** When multiple viable approaches exist (atomic vs. workflow, new skill vs. extension), state the options and tradeoff. A default is fine; the alternative must be named.
- **Push back when the user is likely wrong.** If a request would break a documented convention (skipping the Post-Change Checklist, mixing workflow steps into an atomic skill, em dashes), say so before acting.
- **Verify after editing.** Re-read the changed section, check cross-references resolve, confirm the Post-Change Checklist is addressed. Work isn't done until verified.

## Adding a New Skill

1. Create `plugins/<stack>/skills/<skill-name>/SKILL.md` with the frontmatter above.
2. Workflow skills: prefix `task-`, set `user-invocable: true`, `type: workflow`. Atomic skills: set `user-invocable: false`.
3. Write the body following the standards below.
4. Update the plugin's `README.md` skill table.

### Composition Contracts

- **Workflow skills must load `Use skill: behavioral-principles` as Step 1**, before any other delegation. Universal and unconditional - the behavioral rules must be in effect for every subsequent step.
- Workflows that need to confirm the project type do so as Step 2 by reading its marker file, and stop when the marker is absent rather than degrading into generic advice.
- Review workflows load `Use skill: review-precondition-check` to resolve the change set, then write the report inline as their final step - the frontmatter block and confirmation line are stated in each workflow's Write Report step, not delegated. Sub-scope reviews invoked as a subagent return findings instead of writing; the parent owns the report.
- Atomic skills that depend on context the workflow gathers declare it at the top of the body with a blockquote (e.g. `> Confirm the project's target platforms first - they decide which guidance applies.`).

### Skill Content Standards

#### Description (frontmatter)

- **Hard cap 150 chars; aim for 100-140.** Descriptions load on every session and trigger `/doctor` truncation past ~1% of the context window.
- **Keyword-dense, not prose.** Lead with a verb (Review/Plan/Detect), pack identifying tokens (framework names, key tools, problem space). The skill picker matches on this text - make it trigger-accurate.
- **Positive framing only.** Describe what the skill does. Move "when not to use", phase enumerations, and anti-pattern walls into the body's `When to Use` / `Patterns` sections.

#### Required Body Sections

**Workflow skills** (`task-*`):

| Section           | Purpose                                                                  |
| ----------------- | ------------------------------------------------------------------------ |
| **When to Use**   | Scope, constraints, when NOT to use                                      |
| **Workflow**      | Numbered steps with `Use skill:` delegations to atomic skills            |
| **Output Format** | Template showing the expected deliverable                                |
| **Self-Check**    | Checkbox list aligned 1:1 with workflow steps                            |
| **Avoid**         | Anti-patterns and common mistakes                                        |

**Atomic skills:**

| Section           | Purpose                                                |
| ----------------- | ------------------------------------------------------ |
| **When to Use**   | Usage scope                                            |
| **Rules**         | Non-negotiable constraints                             |
| **Patterns**      | Detailed guidance with bad/good code pairs             |
| **Output Format** | Structured contract that consuming workflows parse     |
| **Avoid**         | Domain-specific anti-patterns                          |

#### Content Quality Rules

- **Output format is a contract.** Use exact field names and value enums (e.g., `Blast Radius: {Narrow | Moderate | Wide | Critical}`) - workflow skills parse this.
- **Self-check matches workflow steps 1:1.** No checks for steps that don't exist.
- **Bad/good code pairs.** Show the mistake then the fix, both with brief explanation.
- **Tables for decision support** (depth levels, scope options, classification).
- **Handle missing input, unknown stack, partial information explicitly.** Don't fail silently.
- **Workflows compose every relevant atomic skill** via `Use skill:`.
- **Consistent depth across plugins.** Equivalent skills (`flutter-performance` vs. `unity-performance`) cover the same categories.

#### Match the Form to the Failure

Before writing a rule, classify the failure it fixes. The form must match, or the rule underperforms - a prohibition aimed at a shaping failure measurably produces *more* of the unwanted output than saying nothing at all.

| Failure being fixed                                             | Right form                                                          | Wrong form                                     |
| --------------------------------------------------------------- | ------------------------------------------------------------------- | ---------------------------------------------- |
| Agent knows the rule, skips it under pressure                    | Prohibition in `Rules` / `Avoid`                                    | Soft guidance ("prefer...", "consider...")     |
| Output has the wrong shape (malformed, bloated, verdict buried)  | Positive recipe in `Output Format` - state what the output **is**, in order | Prohibition list ("don't restate", "never...") |
| A required element is missing                                    | A required slot in the `Output Format` template                     | Prose reminder near the template               |
| Behavior should vary by situation                                | Conditional keyed to something the agent can observe                | Unconditional rule plus exemption clauses      |

**Routing rule.** Discipline failures go to `Rules` and `Avoid`. Shape failures go to `Output Format` as a positive contract - never to `Avoid`. This repo's recurring defects (enum truncation, missing writer fields, atomic-delegation dead ends, envelope conflicts) are shape failures; adding another `Avoid` bullet does not fix them.

Two corollaries, both of which cost more than they look:

- **No nuance clauses.** "Don't X unless it matters" reopens the negotiation. A single nuance clause appended to a working recipe degrades it from consistent to noisy.
- **Exemption clauses don't scope.** "This limit doesn't apply to code blocks" still suppresses code blocks. Write the conditional as a predicate instead.

Bad - a nine-item denylist for what is one enum rule:

```
## Avoid
- Emitting `[Question]`, `[Suggestion]`, `[Consider]`, `[Nit]`, `[Nitpick]`, or `[Praise]` labels
```

Good - the same rule as a contract, in `Output Format`:

```
## Output Format
Every finding carries exactly one label: `[Must]` or `[Recommend]`. No other label is written.
```

This is a forward-looking convention. Retrofit opportunistically, when a skill is open for another reason.

#### Authoring for Token Efficiency

Skills load into context on every invocation - longer is not better. Optimize up front so skills ship close to their post-eval state.

- **Rewrite, don't patch.** Layered patches accumulate ambiguity. When a section grows unclear, rewrite it.
- **Abstract, don't accumulate.** Three rules saying variations of the same thing collapse to one. Specific cases live in `Patterns`, not `Rules`.
- **Micro-examples beat large examples.** A 3-5 line bad/good pair clarifies more than a 30-line scenario. Keep examples only when they clarify behavior, define output structure, or show a non-obvious convention.
- **No duplication.** Each rule appears once. If it overlaps with `behavioral-principles` or another atomic skill, delete the local copy and let composition handle it.
- **Cut filler.** Drop "this skill helps you...", restated frontmatter, repeated motivation, procedural narration. Every sentence adds new information.
- **No hedging.** Skills are contracts - state the rule or omit it. No "you might want to consider...".
- **Halve any section that doesn't lose meaning when halved.**
- **Don't add unless necessary.** Default to simplify/compress/generalize over append.

## Adding a New Agent

1. Create `plugins/<stack>/agents/<agent-name>.md` with the frontmatter below.
2. Update the plugin's `README.md` agents table.

```yaml
---
name: <stack>-<role>                              # required: kebab-case, matches filename
description: Short description                    # required: shown in agent picker
category: quality | engineering | planning | ops  # optional but encouraged
tools: Read, Write, Edit, Bash, Glob, Grep        # optional: restrict tools
---
```

Only `name` and `description` are required; include the others when they meaningfully constrain the agent.

## Post-Change Checklist

After any change to plugin content (skills, agents, structure, conventions) - **excluding changes that only touch `CLAUDE.md` or `README.md`**:

1. **`CLAUDE.md`** - update if structure, conventions, naming, design principles, or workflow guidance changed.
2. **Root `README.md` and affected plugin `README.md`** - reflect added/removed/renamed skills or agents.
