# stk

Stacked PRs for GitHub, made simple.

`stk` lets you break large changes into small, reviewable PRs that depend on each other — without the pain of manually rebasing, retargeting, and tracking what goes where.

## Why stk

You're building a feature. It touches the data model, then the service layer, then the API. You could open one massive PR and watch your reviewers cry. Or you could stack three small PRs that merge in sequence.

The problem: Git doesn't understand stacks. When PR #1 gets feedback, you fix it, then manually rebase #2 and #3, update their base branches, force-push everything, and pray you didn't mess up. Do this a few times and you'll start avoiding stacked PRs altogether.

`stk` fixes this:

- **One command to sync** — edit any branch, `stk sync` rebases everything above it
- **PRs stay connected** — each PR shows the full stack with status, so reviewers know the context
- **Land with confidence** — `stk land` merges the bottom PR, retargets the next one, and rebases the rest
- **Multiple stacks** — work on several independent stacks at once, each tracked separately

## Installation

```bash
cargo install stk
```

Requires the [GitHub CLI](https://cli.github.com/) (`gh`) installed and authenticated.

## Quick Start

You're on `main`. Time to build a feature in two stacked PRs.

```bash
# Create the first branch in your stack
stk new add-user-model
# ... write code ...
git add -A && git commit -m "feat: add user model"

# Stack another branch on top
stk new add-user-service
# ... write code ...
git add -A && git commit -m "feat: add user service"

# Push the entire stack and create PRs
stk push
```

That's it. Both PRs are created with the correct base branches, and each PR description shows the full stack.

### When you get feedback on the first PR

```bash
# Go back to the first branch
stk switch add-user-model

# Make your changes
git add -A && git commit -m "fix: address review feedback"

# Sync the stack — rebases everything above
stk sync

# Push updated stack
stk push
```

### When the first PR is approved

```bash
# Merge the bottom PR, rebase the rest onto main
stk land
```

### Working with multiple stacks

You can have multiple independent stacks at once. Each stack is identified by its root branch (the bottom one).

```bash
# You're working on the user feature stack
stk new add-user-model
git add -A && git commit -m "feat: add user model"
stk push

# Start a completely separate stack for a bug fix
git checkout main
stk new fix-payment-bug
git add -A && git commit -m "fix: payment validation"
stk push

# See all your active stacks
stk status
```

## Commands

### `stk status`

Show all active stacks with their PR status.

```bash
stk status
```

```
Stacks

  add-user-model → add-user-service → add-user-controller
  #42 ✅ Merged    #43 👈 Open         #44 🔄 Draft

  fix-payment-bug → fix-payment-tests
  #50 🔄 Open       (not pushed)

  refactor-utils
  #55 ✅ Approved
```

### `stk status <stack>`

Show detailed view of a specific stack. The stack is identified by its root branch name.

```bash
stk status add-user-model
```

```
Stack: add-user-model (3 branches)

  main
    ↑
  ● add-user-model (#42) ✅ Merged
    ↑
  ● add-user-service (#43) 🔄 Open  ← you are here
    ↑
  ○ add-user-controller (#44) 📝 Draft
```

### `stk new <name>`

Create a new branch stacked on your current branch.

```bash
stk new my-feature
```

If you're on `main`, this starts a new stack. If you're on a stacked branch, this adds to the stack.

### `stk push`

Push all branches in the current stack and create/update their PRs.

```bash
stk push
```

- Creates PRs with correct base branches (PR #2 targets PR #1's branch, etc.)
- Updates PR descriptions with stack overview
- Pushes with `--force-with-lease` for safety

### `stk sync`

Rebase the entire stack after making changes.

```bash
stk sync
```

If you edited a branch in the middle of the stack, this rebases all branches above it. If `main` has new commits, this rebases the entire stack onto `main`.

**On conflicts:** `stk sync` stops and lets you resolve. After fixing:

```bash
git add -A
stk sync --continue
```

### `stk switch <branch>`

Switch to any branch, in any stack.

```bash
stk switch add-user-service
```

This is a convenience wrapper around `git checkout` that works across all your stacks.

### `stk land`

Merge the bottom PR and rebase the stack onto `main`.

```bash
stk land
```

This:

1. Squash-merges the bottom PR via GitHub
2. Retargets the next PR to `main`
3. Rebases remaining branches onto `main`
4. Updates all PR descriptions

## How it works

`stk` stores stack information in two places:

### 1. Visible footer in PR descriptions

Every PR in a stack gets a footer that reviewers can see:

```markdown
---

**Stack** (2 of 3) — merge bottom to top

| PR                             | Status     |
| ------------------------------ | ---------- |
| #42 feat: add user model       | ✅ Merged  |
| **#43 feat: add user service** | 👈 This PR |
| #44 feat: add user controller  | 🔄 Open    |

<sub>Managed by stk</sub>
```

This keeps reviewers informed without requiring them to install anything.

### 2. Hidden metadata for stk

Below the visible footer, `stk` adds an HTML comment with machine-readable data:

```html
<!-- stk
{
  "version": 1,
  "stack_id": "a1b2c3",
  "position": 2,
  "branches": [
    "add-user-model",
    "add-user-service",
    "add-user-controller"
  ]
}
-->
```

This allows `stk` to:

- Reconstruct the stack from any machine
- Detect merged PRs (via GitHub's GraphQL API)
- Keep all PRs in sync when the stack changes

### Detecting merged PRs

When you use squash-merge, the original commits disappear from history. `stk` detects merged PRs by querying GitHub's `associatedPullRequests` API, which links squashed commits back to their original PR.

## Requirements

- **GitHub** — `stk` is built for GitHub (GitLab support may come later)
- **GitHub CLI** — `gh` must be installed and authenticated
- **Squash merge** — `stk` assumes your repo uses squash merging
