# Section 12: Git Workflow & Code Review

## Goal

Define how code moves from a developer's machine to production. Strict, predictable, low-ceremony. Every team member should know exactly which branch to start from, how to name it, how to write commits, what gets merged where, and what gates a PR must pass.

## Branching model

**Environment branching** — three long-lived branches that each mirror a deployment environment.

| Branch | Purpose | Deploys to |
|---|---|---|
| `dev` | Latest work in progress; may be unstable | dev environment |
| `staging` | Release candidate under QA | staging environment |
| `main` | What's running in production | production environment |

### Forward flow (normal feature)

```
feature/order-placement   (branched from dev)
        │
        ▼  (PR — reviewed, CI green)
       dev                              [deployed to dev environment]
        │
        ▼  (PR — when ready for QA, periodic)
     staging                            [deployed to staging environment, QA tests]
        │
        ▼  (PR — when QA passes, scheduled releases)
       main                             [deployed to production]
```

### Hotfix flow (urgent production bug)

```
hotfix/critical-payment-bug   (branched from main)
        │
        ▼  (PR — fast-track review)
       main                             [deployed to production immediately]
        │
        │  (back-merge — MANDATORY within 24h)
        ▼
     staging
        │
        │  (back-merge — MANDATORY within 24h)
        ▼
       dev
```

The two back-merges after a hotfix are **mandatory** to prevent the bug returning when `dev` → `staging` → `main` rolls forward again.

## Branch naming

Prefix every branch with one of:

| Prefix | Use case |
|---|---|
| `feature/` | New functionality |
| `fix/` | Non-urgent bug fix |
| `hotfix/` | Urgent production bug that branches from `main` |
| `chore/` | Maintenance, refactoring, dependency upgrades, doc-only changes |

Rules:
- Lowercase, hyphen-separated
- Short and descriptive — `feature/order-placement`, not `feature/new-stuff`
- No ticket IDs in branch names

Examples:
```
feature/order-placement
fix/fixing-order-number-issue
hotfix/critical-payment-bug
chore/upgrade-laravel-12
```

### Which branch to branch FROM

| Type of work | Branch from |
|---|---|
| Feature, fix, chore | `dev` (latest) |
| Hotfix | `main` (current production) |

```bash
# Normal feature
git checkout dev && git pull
git checkout -b feature/order-placement

# Hotfix
git checkout main && git pull
git checkout -b hotfix/critical-payment-bug
```

## Commit messages — Conventional Commits

All commits follow the [Conventional Commits](https://www.conventionalcommits.org/) format:

```
<type>(<optional scope>): <short description>

<optional body>

<optional footer>
```

### Types

| Type | Use case |
|---|---|
| `feat` | New feature |
| `fix` | Bug fix |
| `chore` | Maintenance — deps, build config, refactor with no behavior change |
| `refactor` | Code restructure without behavior change |
| `test` | Adding or updating tests |
| `docs` | Documentation only |
| `perf` | Performance improvement |
| `ci` | CI/CD changes |
| `build` | Build system or external dependency changes |
| `style` | Code style only (formatting, semicolons) — rare given Pint runs in CI |

### Format rules

- Type lowercase
- Imperative present tense: "add" not "added" or "adds"
- No period at the end of the subject line
- Subject line under 72 characters
- Body, if present, wraps at 72 characters and explains the *why*, not the *what*

### Examples

```
feat: add order placement endpoint
fix: prevent duplicate orders on rapid submit
chore: upgrade laravel-data to 4.x
refactor(orders): extract loyalty calculation into a Service
test: add coverage for CancelOrder action
docs: clarify FormRequest validation rule
perf: eager-load order items in dashboard query
ci: add Postgres service to test workflow

feat(payments): add Stripe refund flow

Adds RefundOrder action and StripeService::refund() method.
Translates Stripe card exceptions into PaymentDeclinedException.
```

### Scope is optional but useful

Use a scope when the change is clearly within one domain — `feat(orders):`, `fix(auth):`. Skip the scope when the change spans multiple areas.

## Merge strategy

**Merge commit for every PR.** GitHub setting: "Allow merge commits"; disable squash and rebase.

This preserves the full history of every branch and shows in the graph exactly when and what was merged from where. The tradeoff is a busier history; that's accepted.

```bash
# When merging via GitHub UI:
# Choose "Create a merge commit"
```

The merge commit's auto-generated message follows this shape — keep it:

```
Merge pull request #42 from <user>/feature/order-placement

feat: add order placement endpoint
```

## Pull request rules

### Required gates before merging

Every PR must satisfy:

1. **At least 1 approving review** from another team member
2. **All CI checks green** — Pint, Larastan, Rector, tests, coverage
3. **Conversation resolved** — all review comments addressed (either fixed or explicitly acknowledged)

### PR description

A PR description is **required for all PRs except hotfixes**. Use this template (lives in `.github/PULL_REQUEST_TEMPLATE.md`):

```markdown
## Description
What does this PR do? Why is it needed?

## What's new
- Bullet list of meaningful changes only
- Skip trivial items (typos, formatting, dep bumps already in title)

## Notes
- Anything reviewer should pay attention to
- Trade-offs taken, alternatives considered
- Migration steps if any
```

Hotfix PRs may skip the template but must include at minimum a one-line description of what was broken and what was fixed.

### Size guidance

Keep PRs small enough that one reviewer can meaningfully understand them. There is no hard size limit. As a soft norm: PRs over ~500 lines of diff are likely doing too much and benefit from being split. Reviewers may ask large PRs to be split before approving.

## Branch protection

Configured via GitHub branch protection on all three long-lived branches (`dev`, `staging`, `main`).

Per branch:
- ✅ Require pull request before merging
- ✅ Require 1 approval
- ✅ Require status checks to pass before merging (all CI checks)
- ✅ Require branches to be up to date before merging
- ✅ Block force pushes
- ✅ Block deletions

This is GitHub's standard protection setup, applied uniformly to all three branches.

## Rules

1. **No direct pushes to `dev`, `staging`, or `main`.** Always via PR.
2. **Branch from `dev` for everything except hotfixes.** Hotfixes branch from `main`.
3. **Hotfix back-merges within 24 hours.** `main` → `staging`, then `staging` → `dev`. No exceptions.
4. **One PR, one purpose.** Don't bundle unrelated changes.
5. **Conventional Commit messages on every commit.** Not just PR titles.
6. **PR template required** for all PRs except hotfixes.
7. **No merging your own PR without an approving review** (even as the repo owner; ask a teammate, even briefly).
8. **Don't approve a PR you didn't actually review.** Drive-by approvals defeat the purpose.
9. **CI must be green.** No merging with red checks. No "fix in a follow-up" — fix it before merging.
10. **Delete feature branches after merge.** GitHub does this automatically; enable it.
11. **Never rewrite history on `dev`, `staging`, or `main`.** No `git push --force`. Branch protection enforces this.
12. **Feature branches may be force-pushed by their owner** before merge (rebase, cleanup). Once a PR has reviews, prefer additional commits to force pushes.

## Onboarding for new developers

When a new developer joins:

1. Clone the repo. `dev` is the default branch — they branch from there.
2. They read `.github/PULL_REQUEST_TEMPLATE.md` to see the PR shape.
3. They make their first PR small (typo fix, small refactor) to walk through the workflow.
4. They learn the hotfix back-merge rule explicitly — this is the one rule easy to forget.

## What this section deliberately doesn't include

- **No code review checklist.** Team trust is the standard; reviewers apply judgment from the rest of these docs.
- **No release / tagging convention.** Production deploys are triggered by merges to `main`, not by tags. Versioning (semver tags) will be addressed in Section 13 (CI/CD) or 14 (Deployment).
- **No changelog automation.** Conventional Commits supports it; if added later, it'll be in Section 13.
- **No commit signing requirement.** Considered overkill at current team size; can be revisited.
