---
name: git-commit-messages
description: Generate Conventional Commits messages from the current git diff. Use whenever the user asks to write a commit, a commit message, or says "commit this", "based on the diff", "stage and commit" — even if they don't say "Conventional Commits". Outputs a ready-to-paste git commit -m command and never runs it.
---

# Git Commit Messages

## Overview

Turn the current diff into a commit message the user can paste straight into their terminal. Read what actually changed, work out the single idea behind it, then write a Conventional Commits message. The output is a command, not an action — you never run `git commit` yourself.

## When to Use

- The user asks for a commit message, a commit, or "write the commit for the current diff".
- The user says "commit this", "stage and commit", or "what should this commit say?".
- You finish a coding task and the project convention is to close with a commit message (see the repo's `AGENTS.md`).

**When NOT to use:**

- The user only wants a code review, a diff summary, or a changelog entry.
- The working tree is clean — say so instead of inventing a message.
- The user explicitly asked you to make the commit — then still write the message first, but you may run it if they asked.

## Workflow

### 1. Inspect the diff

Start broad, then zoom in. The goal is to understand scope before writing a word.

```bash
git status
git diff --cached --stat
git diff --stat
```

- Prefer the **staged** diff (`--cached`) when anything is staged — that is what will be committed.
- Fall back to the **unstaged** diff when nothing is staged.
- If `git status` shows a clean tree, stop and tell the user there is nothing to commit.

Then read the full diff (`git diff --cached` or `git diff`). The `--stat` only tells you which files moved; the hunks tell you what the change means.

### 2. Classify the change

Pick one **type** and one **scope**. If the diff genuinely contains more than one unrelated concern, do not force it into one message — propose a split (see below).

### 3. Draft the message

Write the subject first, then add body lines only when they earn their place. The body explains **what** changed in more detail and, more importantly, **why**.

### 4. Emit the command

Output a single fenced `bash` block containing the `git commit` command. Use **exactly one** plain `-m "..."` flag holding the whole message — no `$'...'` quoting, no `\n` escapes. Never chain `-m -m -m`. Never execute it, never stage files, never print commentary inside the block.

## Conventional Commits Format

```
<type>(<scope>): <subject>
```

One line, always. No body, no footer.

### Types

| Type | Use for |
|------|---------|
| `feat` | A new feature or capability visible to users |
| `fix` | A bug fix |
| `docs` | Documentation only |
| `style` | Formatting, white space, missing semi-colons — no behaviour change |
| `refactor` | A code change that neither fixes a bug nor adds a feature |
| `perf` | A change that improves performance |
| `test` | Adding or correcting tests |
| `build` | Build system or dependency changes |
| `ci` | CI configuration and scripts |
| `chore` | Other maintenance that does not touch src or test |
| `revert` | Reverting a previous commit |

### Rules

- **Subject**: imperative mood ("add", not "added" or "adds"), lower-case, no trailing full stop, 72 characters or fewer.
- **Scope**: the module, area or file group affected — `auth`, `dashboard`, `api`, `deps`. Omit it only when the change is genuinely repo-wide.
- **One line only**: always emit a single flat subject line, e.g. `git commit -m "feat(search): add incremental result filtering"`. Never add a body, never use `$'...'` quoting, never embed `\n`.
- **Footer**: not included. If a `BREAKING CHANGE` note is truly needed, say so in a short sentence of commentary outside the code block.

The subject should name the area or general idea changed so it stands on its own later.

## Scaling by Change Size

Match the length to the change, but the message is always a **single line**. No body, ever.

| Change | Shape |
|--------|-------|
| One file, trivial (typo, one-liner) | subject only |
| Focused multi-file change | subject only |
| Large or multi-area change | subject only |
| Mixed unrelated concerns | propose split commits, one message each |

## UK Spelling

The user writes in UK English, so the message must too. This applies to the subject, body and scope.

- `initialise`, not `initialize`
- `colour`, `behaviour`, `organisation`, `centre`
- `optimise`, `normalise`, `serialise`
- Use British forms even when the code identifiers use American spelling — the message is prose, the code is not.

## Output Format

Emit exactly one fenced `bash` block with a single plain `-m "..."` flag:

```bash
git commit -m "feat(search): add incremental result filtering"
```

Always a plain quoted one-liner. Do not run the command. Do not add `git add` unless the user asked you to stage. Do not wrap the message in a here-doc or use `-F`. Do not use `$'...'` quoting or `\n` escapes. Do not split the message across multiple `-m` flags under any circumstances.

## Splitting Unrelated Changes

A commit should do one thing. If the diff mixes concerns, say so and propose the split with a message for each, rather than bundling them.

Signals the diff should be split:

- A feature change alongside an unrelated refactor.
- Formatting churn mixed into a logic change.
- Two independent bug fixes touching different modules.
- Dependency bumps bundled with source changes.

## Examples

**Example 1 — trivial fix**
Input: one line in `styles/theme.css` changing a wrong colour variable.
Output:
```bash
git commit -m "fix(theme): correct button hover colour"
```

**Example 2 — focused feature**
Input: added JWT auth across `auth.js` and `routes.js`.
Output:
```bash
git commit -m "feat(auth): implement JWT-based authentication"
```

**Example 3 — larger, multi-file**
Input: reworked a results table into a sortable, paginated component across `table.js`, `table.css` and `theme.css`.
Output:
```bash
git commit -m "feat(table): add sorting and pagination to the results table"
```

**Example 4 — breaking change**
Input: renamed a public function and changed its return shape.
Output:
```bash
git commit -m "refactor(api): rename getTask to getTaskById"
```
Mention the breaking change in a sentence outside the code block if needed.

## Red Flags

- Running `git commit`, `git add`, `git amend`, or `git push` without being asked.
- Inventing changes that are not in the diff, or describing a file that did not change.
- Claiming tests passed, when no tests were run.
- A subject that states the obvious ("update code", "fix stuff", "changes").
- Pasting American spellings into the message.
- Bundling unrelated work into one commit because it happened to be in the same diff.
- Chaining multiple `-m` flags instead of one message.
- Using `$'...'` quoting or `\n` escapes instead of a plain `-m "..."`.
- Adding a body when a single subject line is enough.

## Checklist

Before emitting the command:

- [ ] `git status` confirms there are changes to commit.
- [ ] The staged diff was preferred, or the fallback explicitly noted.
- [ ] The type and scope match the change.
- [ ] The subject is imperative, lower-case, and within 72 characters.
- [ ] The message is a single line with no body or footer.
- [ ] The command is a plain `git commit -m "..."` with no `$'...'` or `\n`.
- [ ] All spelling is UK English.
- [ ] Unrelated concerns were split or flagged.
- [ ] The output is one fenced `bash` block with the command, not an execution.
