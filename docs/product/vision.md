# teenylilqueue — Product Vision (North Star)

## What this repo is
teenylilqueue is a tiny, self-hosted merge queue for GitHub repos, designed for agent-heavy development.

It is pre-production and intentionally simple: install a GitHub App, label PRs, and the queue quietly keeps merges moving.

## The problem
When multiple AI coding agents are working in parallel, they produce PRs that become ready to merge at roughly the same time. Merges then create a churn loop:

- PR A is ready
- PR B merges first
- PR A is now behind the base branch
- PR A rebases, waits for CI, tries again
- PR C merges… repeat

This wastes compute (extra CI runs) and, more importantly, wastes agent time (agents stuck retrying instead of doing useful work).

## North star
**Unblock agents fast without introducing new ceremony.**

Operationally, teenylilqueue exists to minimize the “ready → merged” wall-clock time in repos with lots of concurrent PRs, while staying safe and explainable.

### Success metrics
Primary outcomes we optimize for:

- **Time-to-merge:** median and p95 time from “queued/ready” to “merged”.
- **Rebases avoided:** reduction in “PR is behind base branch” retry loops.
- **CI runs saved:** fewer redundant CI executions caused by rebase churn.

## Primary user
**AI agent ops**: people running many agents, who want merges to happen reliably without babysitting.

A secondary user exists implicitly:

- **The agent author** (human or automated): wants to label a PR and move on.

## Product principles
- **Low ceremony:** no new workflow concepts beyond “label the PR”.
- **Invisible infrastructure:** the queue is background plumbing, not a new product surface.
- **Safe by default:** never bypass GitHub’s branch protection or merge requirements.
- **Transparent state:** always show “what is happening and why” in GitHub.
- **Composable:** works with existing CI and branch protection rules; doesn’t replace them.
- **Least privilege:** request the minimum permissions that still let the queue function.
- **Serverless-first:** deployable on Vercel (stateless compute + external state).

## MVP: what v1 must do
### Supported scope
- **GitHub Cloud (github.com)**.
- **Same-repo PRs only** (no fork PR support in v1).
- **Any base branch**, with **separate queues per base branch**.
- **Multi-install:** one deployment can handle many repo installations.

### Queue membership and ordering
We use a label trigger (not GitHub’s native auto-merge) so this stays free and works for GitHub Free private repos.

- A PR is “in the queue” when it has the **`teenylilqueue`** label.
- Queue ordering is **first-enqueued wins** (timestamp when the label was added).
- Removing the label (or closing the PR) removes it from the queue.
- Draft PRs are **ignored until marked ready**.

### Merge behavior (high level)
When a PR reaches the front of its base-branch queue:

1. **Sync the PR to its base** using GitHub’s **“Update branch”** behavior.
2. **Wait for required checks** to complete on the updated head SHA.
3. If checks pass and GitHub reports the PR is mergeable, **merge via the GitHub API**.
4. If the PR cannot be updated cleanly (conflict) or fails checks, **skip it temporarily** (move it to the back) and continue.
5. Skipped PRs are retried **only on a new push** (head SHA changes).

### Visibility
- All day-to-day visibility is via a **GitHub Check Run** named **`teenylilqueue`**.
- The check details must clearly show:
  - queue position
  - current state (Queued, Updating, Waiting CI, Merging, Skipped, Done)
  - why it is blocked/skipped (conflict, failing checks, waiting on checks)

### Deployment and state
- **Vercel serverless** deployment is a hard requirement.
- Use **webhooks** as the primary driver.
- A small **optional cron reconciler** is acceptable as a safety net.
- Queue state is stored in **Vercel KV** (or equivalent Redis/KV), not on disk.
- Store **minimal data only**: PR identifiers, base/head SHAs, queue order, timestamps.

## Non-goals (explicitly out of scope for v1)
- **Code review** (approvals, reviewer assignment, policy enforcement beyond existing branch protection).
- **CI orchestration** (running CI; teenylilqueue only waits on results).
- **Non-GitHub SCM** (GitLab/Bitbucket/etc.).
- **Monorepo-specific merge logic**.
- **Complex priority rules** (labels like P0/P1, scoring systems, manual reorder).
- **A web UI/dashboard** (GitHub is the UI for v1).
- **Manual override commands** (no `/skip`, no admin command surface in v1).

## Business and distribution
- **Open source, self-hosted**, intended to stay free to run.
- License: **MIT**.

## Product boundary (what “good” means)
teenylilqueue is “good” when teams can run many agents in parallel and still:

- keep merges flowing,
- keep the base branch green,
- avoid the rebase/CI retry treadmill,
- understand queue state from the PR page without leaving GitHub.
