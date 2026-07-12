# pr-codex-review

Local helper for running several Codex PR reviews, aggregating the results, and posting one friendly GitHub PR comment.

This is a local wrapper around `gh`, `git worktree`, `direnv`, and `codex exec review`. It uses your local GitHub and Codex authentication. It does not ship credentials.

## Install

Via Homebrew:

```bash
brew tap yungweng/tap
brew trust yungweng/tap
brew install pr-codex-review
```

This pulls in `gh`, `jq`, and `direnv` automatically. The Codex CLI must be installed separately:

```bash
brew install --cask codex
```

Or manually from this repo:

```bash
ln -sf "$PWD/bin/pr-codex-review" ~/.local/bin/pr-codex-review
```

Requirements:

```text
gh
git
jq
codex
direnv
```

If `glow` is installed, the final comment is rendered in the terminal with it; otherwise it is printed as plain text.

## Normal Run

From inside the repo checkout, the PR number is enough:

```bash
pr-codex-review 123
```

This runs 6 reviewers in parallel, aggregates their findings, posts the comment to the PR, and removes the temporary worktree. A full PR URL still works:

```bash
pr-codex-review https://github.com/owner/repo/pull/123
```

By default, reviewers and the aggregator use `gpt-5.6-terra` with `medium` reasoning effort, regardless of your personal Codex model and effort defaults.

Use `--dry-run` to inspect the comment before anything touches GitHub:

```bash
pr-codex-review 123 --dry-run
```

Pick the reviewer count, model, and reasoning effort per run:

```bash
pr-codex-review 123 -n 8 --model gpt-5.4-mini --effort low
```

## What It Does

1. Reads PR metadata with `gh`.
2. Fetches the PR base branch.
3. Fetches the PR head through `refs/pull/<number>/head`.
4. Creates a detached Git worktree under `~/.cache/pr-codex-review/`.
5. Runs `direnv allow`.
6. Runs `codex exec review --base origin/<base-branch>` several times.
7. Stores each reviewer output.
8. Aggregates the outputs into one PR comment and prints a preview.
9. Posts the comment (unless `--dry-run` is set), writes a machine-readable `findings.json`, and cleans up the worktree.

## Options

```text
-n, --runs N
Number of Codex reviewer passes. Default: 6.

--concurrency N
Maximum number of reviewer passes running at the same time. Default: same as --runs.

--model MODEL
Model for the Codex reviewers and the aggregator. Default: gpt-5.6-terra.

--effort LEVEL
Reasoning effort for the Codex reviewers and the aggregator.
One of: minimal, low, medium, high, xhigh. Default: medium.

--base BRANCH
Base branch to review against. Default: the PR base branch.

--dry-run
Write the aggregated comment to disk without posting it to the PR.

--review-timeout DURATION
Kill a reviewer that runs too long. Default: 45m.
Use `0` to disable. Supports values like `30m`, `45m`, `1h`, or raw seconds.

--min-successful N
Minimum reviewer outputs required to aggregate/post.
Default: majority of the requested runs (4 of 6, 4 of 7, ...).
Use `--min-successful 6` with `--runs 6` to require every reviewer.

--resume-run DIR
Reuse an existing run directory and only aggregate/post.
Use this when reviewer runs finished but aggregation or posting stopped.

--keep-worktree
Keep the temporary worktree after a successful run.
Failed runs always keep the worktree so --resume-run can reuse it.

--no-direnv
Skip direnv.

--allow-envrc-change
Allow automatic direnv allow even when the PR changed `.envrc`.
Use this only after manually checking the `.envrc` diff.

--version
Show version.

-h, --help
Show command help.
```

Deprecated no-ops, kept so old invocations do not break (they describe the default behavior now):

```text
--post
--cleanup
--allow-base-drift
```

## Resume After A Stop

If the runner prints a `Run dir:` and stops after the reviewers finished, reuse that directory:

```bash
pr-codex-review 123 --resume-run ~/.cache/pr-codex-review/owner-repo-pr-123-YYYYMMDD-HHMMSS
```

This does not rerun the expensive reviewer passes. Failed runs keep their worktree, so resuming always works until the run completes successfully.

## Safety Stops

The runner refuses to post if the PR head changed during review. That means the PR received a new commit, so old review results may no longer match the diff.

If only the base branch moved during review, the runner continues and the final comment includes a note with the reviewed base SHA and the latest base SHA.

The runner also refuses automatic `direnv allow` if the PR changed `.envrc`, unless you pass `--allow-envrc-change`.

## Output Files

Each run writes files under:

```text
~/.cache/pr-codex-review/<repo>-pr-<number>-<timestamp>/output/
```

The `output/` directory always survives; only the `worktree/` directory is removed after a successful run (keep it with `--keep-worktree`). Run directories older than 7 days are deleted automatically at the start of the next run.

Useful files:

```text
reviewer-1.md
reviewer-2.md
...
all-reviewers.md
final-pr-comment.md
findings.json
aggregator.log
```

## Machine-Readable Findings

The aggregated comment always uses exactly five sections (`## Summary`, `## Blockers`, `## Critical`, `## Suggestions`, `## Questions`) with one top-level bullet per finding. The runner validates this structure and retries the aggregation once if it is violated; a run that still cannot produce it fails instead of posting.

Next to the comment, each successful run writes `findings.json` with per-section counts, so scripts (for example an automated fix loop) can branch on the result without parsing Markdown:

```json
{
  "schema": 1,
  "pr": 123,
  "head_sha": "…",
  "base_sha": "…",
  "reviewers_succeeded": 6,
  "reviewers_requested": 6,
  "blockers": 0,
  "critical": 2,
  "suggestions": 3,
  "questions": 1,
  "comment_file": "…/output/final-pr-comment.md",
  "posted": true,
  "comment_url": "https://github.com/owner/repo/pull/123#issuecomment-…"
}
```

`comment_url` is `null` on `--dry-run` runs.

View the final comment again later:

```bash
RUN=$(ls -td ~/.cache/pr-codex-review/owner-repo-pr-123-* | head -1)
glow "$RUN/output/final-pr-comment.md"
```

## Quick Commands

Post (default):

```bash
pr-codex-review 123
```

Dry run:

```bash
pr-codex-review 123 --dry-run
```

Resume and post:

```bash
pr-codex-review 123 --resume-run <RUN_DIR>
```
