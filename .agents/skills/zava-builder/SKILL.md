---
name: zava-builder
description: >
  Use this skill when given a work item — a feature brief, a list of review findings, a
  ticket, acceptance criteria, or diff-level feedback — and tasked with implementing it
  in the zava-storefront repo. Triggers on: "implement this", "fix these findings", "build
  this feature", "apply these review comments", "create a PR for", "open a pull request
  for", or whenever the user pastes a ticket or feedback list expecting code output. The
  skill reads /guidelines before writing code, stays in scope, runs npm run lint and npm
  test, fixes failures, then commits and opens or updates the pull request. Does NOT handle
  multi-service architecture decisions or changes requiring an ADR; surfaces those in the
  PR description for a human.
---

# zava-builder

Implement a work item on a new branch, run CI, and open or update a pull request on
zava-storefront. All code written follows the team's guideline files. Anything outside
the work item's scope is noted in the PR description for a human rather than built.

## Prerequisites

Confirm these tools are available before starting. If any are absent, stop and tell the
user which tool is missing.

```
git --version
npm --version
gh auth status
```

## Stage 1 — Plan

### 1.1 Load guidelines (lazy — load only now, not at session start)

Read the following files in order. Extract the rules that are relevant to the work item
type (feature add, refactor, bug fix, security fix). Do not inline the full content into
context; retain only the rules that will govern this work item.

- `guidelines/security.md`   — input validation, secrets, authN/authZ, crypto, logging
- `guidelines/architecture.md` — layer boundaries, module boundaries, ADR requirement
- `guidelines/documentation.md` — docstring format, changelog requirement, doc structure

### 1.2 Parse the work item

Extract from the work item:
- **Goal** — one sentence describing the desired outcome.
- **Tasks** — ordered list of the smallest independently committable changes that together
  achieve the goal.
- **Acceptance criteria** — explicit or inferred from the brief.

### 1.3 Scope-check every task

For each task, apply this rule:

> KEEP if: the task is directly required by the work item AND fits within a single service
> or module boundary without requiring a new external dependency or a cross-service schema
> change.
>
> NOTE-ONLY if: the task requires an ADR, touches multiple services, introduces a new
> external dependency, or is a quality improvement beyond the stated scope.

Assign each task a status: `in-scope` or `noted-for-human`.

### 1.4 Write plan.md (B4 PLAN MEMENTO)

Write a `plan.md` file in the current working directory with this structure:

```
# Build Plan — <goal>

## Goal
<one sentence>

## Acceptance criteria
- <criterion>

## Tasks
- [ ] <task 1> [in-scope]
- [ ] <task 2> [in-scope]
- [ ] <task 3> [noted-for-human: reason]

## Scope notes (for PR description)
- <anything noted-for-human with reason>

## Branch
<branch-name>   (format: feat/<slug> or fix/<slug> or chore/<slug>)
```

### 1.5 Create the branch

```bash
git checkout main
git pull origin main
git checkout -b <branch-name>
```

Verify the branch exists:

```bash
git branch --show-current
```

The output must match `<branch-name>`. If it does not, stop and report the error.

---

## Stage 2 — Implement

For each `in-scope` task in `plan.md`, follow this loop exactly:

### 2.1 Reload plan (B8 ATTENTION ANCHOR)

Re-read `plan.md` before starting each task. Confirm which task is next. Do not rely on
session memory across tasks.

### 2.2 Write the code

Apply the relevant rules loaded from guidelines in Stage 1:

- Layer boundaries from `architecture.md` (no cross-layer imports; infra depends on
  domain, never the reverse).
- Input validation at every trust boundary from `security.md`.
- Parameterize all queries; encode output for destination context.
- Docstrings on every exported symbol per `documentation.md` format (language-appropriate:
  Google style for Python, TSDoc for TypeScript).
- Update `CHANGELOG.md` if the task is a user-facing change.

Do not add scope that is not in `plan.md`. If you notice a problem outside the task's
boundary, add it to the `noted-for-human` section of `plan.md` and continue.

### 2.3 Commit

```bash
git add -A
git diff --staged --stat
git commit -m "<type>(<scope>): <summary>"
```

Commit message format: Conventional Commits.
`type` is one of: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `style`.

Verify the commit landed:

```bash
git log --oneline -1
```

### 2.4 Mark task done in plan.md

Update the task checkbox from `[ ]` to `[x]` in `plan.md`. Write the updated file.

Repeat 2.1–2.4 for each remaining `in-scope` task.

---

## Stage 3 — CI Gate (S4 Validation Decorator)

This stage blocks Stage 4. Do not open the PR until both checks are green.

### 3.1 Run lint

```bash
npm run lint 2>&1
```

If lint exits non-zero, fix all reported violations, then re-run. Only auto-fixable
violations (formatting, unused imports, simple style) should be auto-fixed. If a violation
requires a design decision (e.g. a rule about module boundaries), add it to
`noted-for-human` in `plan.md` and suppress the specific rule with an inline comment
that cites the reason. Do not suppress rules silently.

Commit any auto-fixes:

```bash
git add -A && git commit -m "chore: fix lint violations"
```

### 3.2 Run tests

```bash
npm test 2>&1
```

If tests exit non-zero:
1. Read the failing test names and error output.
2. Determine whether the failure is caused by this branch's changes or was pre-existing.
   - Pre-existing: add to `noted-for-human`; do not mask by skipping tests.
   - Caused by this branch: fix the implementation or update the test if the test's
     expectation was wrong relative to the acceptance criteria.
3. Re-run `npm test` after fixing. Repeat until green or until you determine the failure
   is pre-existing (then note it clearly in the PR description).

Commit any test fixes:

```bash
git add -A && git commit -m "test: fix failing tests"
```

### 3.3 Confirm green

```bash
npm run lint 2>&1 && npm test 2>&1
echo "CI result: $?"
```

The PR stage begins only if the exit code is 0.

---

## Stage 4 — Pull Request

### 4.1 Push the branch

```bash
git push -u origin <branch-name>
```

### 4.2 Compose the PR body

Load `assets/pr-template.md`. Fill it with:
- Summary of what was implemented.
- Links to the acceptance criteria.
- The `noted-for-human` items from `plan.md` under the "Out of scope" section.
- CI status (both checks green).

### 4.3 Open or update the PR

Check whether a PR already exists for this branch:

```bash
gh pr view --json number,url 2>/dev/null
```

If no PR exists:

```bash
gh pr create \
  --title "<type>(<scope>): <goal>" \
  --body-file <(cat pr-body.md) \
  --base main
```

If a PR already exists (update scenario):

```bash
gh pr edit --body-file <(cat pr-body.md)
```

### 4.4 Report to user

Output exactly:

```
PR: <url>
Branch: <branch-name>
Checks: lint OK, tests OK
Out-of-scope notes: <count> item(s) in PR description
```

If there are `noted-for-human` items, list them after the summary so the user sees them
immediately without opening the PR.

---

## Constraints

- Never commit secrets, tokens, or credentials (security.md §2).
- Never concatenate user input into shell commands. Use `--` argument separators and
  quoted variables.
- Never force-push. Never amend a commit after it has been pushed.
- Never skip or suppress a test without an inline comment and a `noted-for-human` entry.
- Never open a PR if CI is red. Fix first.
- If at any point a tool call fails with an unrecoverable error, stop and report the
  exact error output to the user. Do not continue silently.
