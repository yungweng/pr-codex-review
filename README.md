# pr-codex-review

Local helper for running several Codex PR reviews, aggregating the results, and optionally posting one friendly GitHub PR comment.

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

## Normal Run

Use this when you want 6 reviewers in parallel and want the final comment posted automatically:

```bash
pr-codex-review https://github.com/owner/repo/pull/123 \
  --runs 6 \
  --concurrency 6 \
  --allow-base-drift \
  --post
```

Without `--post`, the command is a dry run. It writes the final Markdown comment to disk but does not touch GitHub.

## What It Does

1. Reads PR metadata with `gh`.
2. Fetches the PR base branch.
3. Fetches the PR head through `refs/pull/<number>/head`.
4. Creates a detached Git worktree under `~/.cache/pr-codex-review/`.
5. Runs `direnv allow`.
6. Runs `codex exec review --base origin/<base-branch>` several times.
7. Stores each reviewer output.
8. Aggregates the outputs into one PR comment.
9. Posts the comment only when `--post` is set.

## Options

```text
--runs N
Number of Codex reviewer passes. Default: 6.

--concurrency N
Number of reviewer passes to run at the same time. Default: 2.

--base BRANCH
Base branch to review against. Default: the PR base branch.

--post
Post the aggregated comment to the PR. Without this, it only writes files.

--allow-base-drift
Allow aggregation/posting if only the base branch moved during review.
The PR head must still match the reviewed commit.

--resume-run DIR
Reuse an existing run directory and only aggregate/post.
Use this when reviewer runs finished but aggregation or posting stopped.

--cleanup
Remove the temporary worktree after the run.

--no-direnv
Skip direnv.

--allow-envrc-change
Allow automatic direnv allow even when the PR changed `.envrc`.
Use this only after manually checking the `.envrc` diff.

-h, --help
Show command help.
```

## Resume After A Stop

If the runner prints a `Run dir:` and stops after the reviewers finished, reuse that directory:

```bash
pr-codex-review https://github.com/owner/repo/pull/123 \
  --resume-run ~/.cache/pr-codex-review/owner-repo-pr-123-YYYYMMDD-HHMMSS \
  --allow-base-drift \
  --post
```

This does not rerun the expensive reviewer passes.

## Safety Stops

The runner refuses to post if the PR head changed during review. That means the PR received a new commit, so old review results may no longer match the diff.

The runner also refuses automatic `direnv allow` if the PR changed `.envrc`, unless you pass `--allow-envrc-change`.

If only the base branch moved during review, pass `--allow-base-drift` to continue. The final comment includes a note with the reviewed base SHA and latest base SHA.

## Output Files

Each run writes files under:

```text
~/.cache/pr-codex-review/<repo>-pr-<number>-<timestamp>/output/
```

Useful files:

```text
reviewer-1.md
reviewer-2.md
...
all-reviewers.md
final-pr-comment.md
aggregator.log
```

View the final comment:

```bash
RUN=$(ls -td ~/.cache/pr-codex-review/owner-repo-pr-123-* | head -1)
glow "$RUN/output/final-pr-comment.md"
```

## Quick Commands

Dry run:

```bash
pr-codex-review <PR_URL> --runs 6 --concurrency 6 --allow-base-drift
```

Post:

```bash
pr-codex-review <PR_URL> --runs 6 --concurrency 6 --allow-base-drift --post
```

Resume and post:

```bash
pr-codex-review <PR_URL> --resume-run <RUN_DIR> --allow-base-drift --post
```
