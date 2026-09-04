---
name: babysit-pr
description: Babysit a GitHub pull request after creation by continuously polling review comments, CI checks/workflow runs, and mergeability state until the PR is merged/closed or user help is required. Diagnose failures, retry likely flaky failures up to 3 times, auto-fix/push branch-related issues when appropriate, and keep watching open PRs so fresh review feedback is surfaced promptly. Use when the user asks Codex to monitor a PR, watch CI, handle review comments, or keep an eye on failures and feedback on an open PR.
---

**Cross-repository work:** If scope spans repositories, invoke `$graphify` before discovery, planning, or edits. Query an existing graph; build/update a merged graph when missing, stale, or incomplete. Reuse a current graph for the same repository set.

# PR Babysitter

Babysit until the PR is merged/closed, or user help is required (CI infra, flaky retries exhausted, permissions, unsafe ambiguity). Green + mergeable + review-clean is a progress milestone, not a stop. Do not stop on a single `idle` snapshot while checks are pending.

Inputs: none (`--pr auto` from current branch), PR number, or PR URL.

## Commands

```bash
python3 .skills/babysit-pr/scripts/gh_pr_watch.py --pr auto --once
python3 .skills/babysit-pr/scripts/gh_pr_watch.py --pr auto --watch
python3 .skills/babysit-pr/scripts/gh_pr_watch.py --pr auto --retry-failed-now
python3 .skills/babysit-pr/scripts/gh_pr_watch.py --pr <number-or-url> --once
```

Prefer `--watch` for monitor/watch/babysit. Use `--once` only for diagnostics, local testing, or an explicit one-shot. Keep exactly one watcher for this PR/state file in this turn; consume output instead of detaching.

## Loop

On each snapshot:

1. Merged/closed (`stop_pr_closed`): report the terminal state and stop immediately.
2. Inspect `actions`. Handle new review items before CI or mergeability work. Verify mergeability (for example `gh pr view`) alongside CI.
3. `process_review_comment`: handle reviews below.
4. `diagnose_ci_failure`: inspect failed-run logs and classify (see heuristics).
5. Branch-related: patch, commit, push on the PR head, then resume watch on the new SHA.
6. Flaky/unrelated and `retry_failed_checks`: `--retry-failed-now` (default 3 cycles per SHA). If a review fix needs a commit, do that first and skip rerun on the old SHA.
7. After push, rerun, reply, or resolve: brief progress update, then continue. If `--watch` was paused for a patch, relaunch it in the same turn. A push is not a stop.
8. Green + mergeable + review-clean: report ready-to-merge and keep watching for late reviews/conflicts.
9. User-help blocker: report and stop.
10. Otherwise keep polling. Do not ask whether to continue.

## Reviews

Watcher surfaces issue comments, inline comments, and review submissions, including Codex bot `chatgpt-codex-connector[bot]`. It auto-surfaces trusted humans (OWNER/MEMBER/COLLABORATOR, authenticated operator) and approved review bots; ignore unrelated bot noise. A fresh state file may include already-pending feedback.

Actionable and correct: patch; commit `codex: address PR review feedback (#<n>)`; push the PR head; mark the thread/comment resolved; resume `--watch` immediately.

Disagree, non-actionable, or already addressed: one GitHub reply prefixed `[codex]`. Ignore later self-authored items. Ignore already-resolved threads unless new unresolved follow-up appears.

## CI

```bash
gh run view <run-id> --json jobs,name,workflowName,conclusion,status,url,headSha
gh run view <run-id> --log-failed
```

Branch-related: compile/test/lint/typecheck/snapshots/static analysis in touched areas. Flaky: timeouts, runner provisioning, registry/network, Actions infra. Ambiguous: one diagnosis attempt before rerun. CI fix commit: `codex: fix CI failure on PR #<n>`.

## Git and permissions

Work only on the PR head. No destructive git. Do not switch branches unless needed to recover context. Unrelated uncommitted changes, `gh` auth/permission failure, or inability to push: stop and ask.

## Cadence

Not green: poll every 1 minute. Green but still open: keep the base cadence for new reviews/conflicts. Reset on SHA, check, review, or mergeability change. Merged/closed: stop immediately.

Keep polling when: `idle` with pending checks; CI running/queued; review quiet but CI not terminal; CI green but mergeability unknown/pending; CI green and mergeable but PR still open; `REVIEW_REQUIRED` (surface new comments, do not ask to keep watching).

## Output

Progress on status changes plus occasional heartbeats. No final summary, and do not end the turn, until a strict stop. When CI first goes all-green for this SHA, one-time: `🚀 CI is all green! 33/33 passed. Still on watch for review approval.`

Final summary: SHA, CI, mergeability/conflicts, fixes pushed, flaky retries used, remaining failures/reviews.

## References

- `.skills/babysit-pr/references/heuristics.md`
- `.skills/babysit-pr/references/github-api-notes.md`
