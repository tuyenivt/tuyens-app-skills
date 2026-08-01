---
name: review-precondition-check
description: Gate code review workflows: resolve the working-tree change set, confirm something is in scope, emit the review handle. Local git only.
metadata:
  category: review
  tags: [review, git, local-git, gating, working-tree]
user-invocable: false
---

# Review Precondition Check

## When to Use

- Step 0 of any code review workflow, before risk or finding analysis.
- Whenever the set of changed files must be resolved and confirmed non-empty before review begins.

This skill **gates only**: it emits the change set and how it was derived, not the diff itself. The consuming workflow runs `git diff` to read content. Review targets the **working tree** - uncommitted changes are the subject, not an obstacle.

## Rules

- **Local git only.** `git status`, `git diff`, `git rev-parse`, `git stash list`. No `gh`, no GitHub MCP, no platform API.
- **No state-changing git commands.** No `fetch`, `checkout`, `stash`, `commit`, `add`. When the user must run one, print the exact command and stop.
- **Stop when there is nothing to review.** An empty change set is a stop, not an empty report.
- **Untracked files are in scope**, but only those the workflow can read as text. Binary and generated paths are listed separately and never diffed.
- **Output a minimal handle.** The change set, the mode that produced it, and notes. The consumer composes its own diff commands.

## Scope Modes

Resolved in order; the first that yields a non-empty change set wins.

| Mode             | Change set                                          | When it applies                                        |
| ---------------- | --------------------------------------------------- | ------------------------------------------------------ |
| `working-tree`   | Unstaged + staged + untracked, vs `HEAD`            | Default. Anything uncommitted exists.                  |
| `staged-only`    | Staged only, vs `HEAD`                              | Workflow passed `--staged`.                            |
| `last-commit`    | `HEAD~1..HEAD`                                      | Tree is clean and the workflow passed no other scope.  |

`last-commit` is the clean-tree fallback so a review run right after committing still has a subject. When `HEAD` has no parent (initial commit), diff against the empty tree: `git diff $(git hash-object -t tree /dev/null) HEAD`.

## Pattern

### Step 1 - Resolve the change set

```bash
git status --porcelain
```

Parse the porcelain output into the change set. Column meanings: position 1 is the staged status, position 2 the unstaged status, `??` is untracked.

- `--staged` passed -> keep entries with a non-space, non-`?` staged column.
- Otherwise -> keep every entry.

If the output is empty, the tree is clean: fall back to `last-commit` mode and resolve the change set from `git diff --name-status HEAD~1..HEAD`.

### Step 2 - Confirm the change set is non-empty

If the change set is still empty after the fallback, stop:

```text
Nothing to review. The working tree is clean and HEAD has no parent commit to compare against.

Make a change, or pass an explicit commit range.
```

### Step 3 - Partition the change set

Split resolved paths into three lists:

- `reviewable` - text files the workflow will diff.
- `binary` - detected via `git diff --numstat` reporting `-` for both counts, or by extension for untracked files (images, audio, archives, fonts, compiled artifacts).
- `generated` - paths matching the project's generated-output conventions (`*.g.dart`, `*.freezed.dart`, `*.designer.cs`, lockfiles, `build/`, `Library/`).

Binary and generated paths are reported as counts in the handle and excluded from `reviewable`. Never diff them.

### Step 4 - Note stashed work

```bash
git stash list
```

If non-empty, add a note: `<N> stash entr(y|ies) present - not included in this review.` This is informational; it never blocks.

### Step 5 - Report scale

Count `reviewable` paths and total changed lines (`git diff --shortstat`, plus untracked file line counts). Record both in the handle so the workflow can size its own effort and decide whether to warn about a large change set.

## Output Format

When preconditions pass, emit this handle and nothing more. Paths are repo-relative. Omit `notes` when empty.

```yaml
review-target:
  mode: working-tree | staged-only | last-commit
  base: <HEAD, HEAD~1, or the empty-tree hash>
  current_branch: <branch short name, or "detached">
  reviewable:
    - <path>
  counts:
    reviewable: <N>
    binary: <N>
    generated: <N>
    changed_lines: <N>
  notes:
    - <stash presence, large change set, binary/generated exclusions, initial-commit fallback>
```

When a precondition fails, emit only the stop message from the relevant step. Do not emit a partial handle.

## Avoid

- Reading file content or computing the diff body - the consuming workflow does that.
- Running any state-changing git command - the user must run these to protect uncommitted work.
- Treating a dirty working tree as a failure - uncommitted changes are the review subject.
- Diffing binary or generated paths.
- Emitting an empty handle instead of the Step 2 stop message.
