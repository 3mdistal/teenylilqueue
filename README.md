# teenylilqueue

**Pre-production:** this project is under active development and not ready for production use.

A tiny, self-hosted merge queue for GitHub repos. Runs on Vercel.

## Why?

When running multiple AI coding agents in parallel, they often create PRs that need to be merged around the same time. This causes a painful loop:

1. Agent A's PR is ready to merge
2. Agent B's PR merges first
3. Agent A's PR is now behind main
4. Agent A rebases, waits for CI, tries again
5. Meanwhile Agent C merged...
6. Repeat forever

**teenylilqueue** solves this by:
- Watching for PRs with auto-merge enabled
- Queuing them in order
- Rebasing each PR onto main when it reaches the front of the queue
- Letting GitHub's auto-merge complete the merge once CI passes

## How It Works

1. Install the GitHub App on your repos
2. Agents enable auto-merge on PRs: `gh pr merge --auto --merge`
3. teenylilqueue handles the rest

Agents are unblocked immediately. No new concepts to learn. The queue is invisible infrastructure.

## Status

Under development. See [Issue #1](../../issues/1) for the implementation plan.
