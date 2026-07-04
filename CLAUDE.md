# CLAUDE.md

Guidance for coding agents working in this repository.

## What this is

A single Bash script (`bin/pr-codex-review`) that runs several parallel `codex exec review` passes against a GitHub PR, aggregates the reviewer outputs with one more Codex call, and optionally posts the result as a PR comment. There is no build step, no package manifest, and no test suite beyond the checks below.

## Layout

```text
bin/pr-codex-review                      the entire tool
README.md                                user-facing docs, keep in sync with --help
.github/workflows/update-homebrew-tap.yml  release automation
```

## Constraints

- The script must stay compatible with the macOS system Bash 3.2. No `mapfile`, no associative arrays, no `${var,,}`. Users may run it via `/bin/bash` even though the shebang prefers `env bash`.
- Runtime dependencies are `gh`, `git`, `jq`, `codex`, and `direnv`. Do not add new ones casually; each one becomes a Homebrew formula dependency (and `codex` is a cask, which formulas cannot depend on).
- The `--help` text, the option parsing, and the README option list describe the same flags. When you touch one, update all three.
- Safety stops are deliberate: refusing to post on PR head drift, refusing automatic `direnv allow` on `.envrc` changes, refusing to post fenced or meta aggregator output. Do not weaken them to make a run pass.

## Checks before committing

```bash
bash -n bin/pr-codex-review
shellcheck bin/pr-codex-review
bin/pr-codex-review --help
```

## Releases

Distribution is a Homebrew tap (`yungweng/homebrew-tap`). The formula there is updated automatically; never edit it by hand for a version bump.

1. Set `VERSION` at the top of `bin/pr-codex-review` to the new version (otherwise `--version` lies).
2. Commit and push to `main`.
3. `git tag vX.Y.Z && git push origin vX.Y.Z`
4. `gh release create vX.Y.Z --title "vX.Y.Z" --notes "..."`

Publishing the release (not the tag) triggers `update-homebrew-tap.yml`, which recomputes the tarball SHA and pushes the bumped formula to the tap using the `TAP_DEPLOY_KEY` secret (a write deploy key on the tap repo).

Version bumps: patch for pure bugfixes, minor for anything that adds or changes a flag.
