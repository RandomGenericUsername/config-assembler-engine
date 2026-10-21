---
id: REQ-001
title: "Birth the config-assembler-engine repo (mu-strict) and cut v0.1.0"
kind: requirement
status: done
lane: 3
links:
  openspec: repo-birth
  bmad: none
worktree: .worktrees/req-001-archive/
branch: ticket/req-001-archive-archive-change
---

# Birth the config-assembler-engine repo (mu-strict) and cut v0.1.0

## Requirement

Slice **S2** of the CLI-repos split SHALL create
`RandomGenericUsername/config-assembler-engine` as a public repo whose root is the imported
`src/shared/config-assembler-engine` package, preserve that package's history with
`git subtree split`, install the `mu` harness byte-for-byte, and cut the repo's first tag
(`v0.1.0`) only after the gates, packaging and CI exist.

The package remains the **tool-agnostic resolution engine**: this slice changes no strategy,
parser, validator, override or merge behaviour, adds no dependency, and adds no sibling-repo
import (spine AD-1, AD-2). `dotfiles-v3` is not modified — the consumer rewire (csg/weg/itr) and
the in-tree deletion are the next step of the slice in that repo.

## Acceptance

- [x] The repo exists, is **public**, and its `main` carries the six imported commits from the
  subtree split.
  Verify: `gh api repos/RandomGenericUsername/config-assembler-engine --jq .private` → `false`;
  `git log --oneline | wc -l` → ≥ 6.
- [x] The harness is byte-identical to `mu` at every copied path.
  Verify: `diff` per path (see the evidence table) → empty, exit 0.
- [x] `AGENTS.md` carries the evidence rule and this repo's charter; `CLAUDE.md` is a symlink.
  Verify: `rg -c "Evidence:|Verify:" AGENTS.md` → ≥ 2; `test -L CLAUDE.md` → exit 0.
- [x] `tickets/INDEX.md` lists this ticket with the same status as the frontmatter, and the
  OpenSpec change validates strictly.
  Verify: `openspec validate repo-birth --strict` → exit 0.
- [x] A fresh clone passes `make sync && make test && make gates && make hermetic` and builds both
  artifacts.
  Verify: `uv sync --frozen && uv run pytest -q && make gates && make build` → exit 0;
  `ls dist/config_assembler_engine-0.1.0-*` → wheel + sdist.
- [x] `.github/workflows/gates.yml` exists and its job set equals `GATES` (plus declared CI-only
  jobs); the first run on `main` is green.
  Verify: `make ci-gates-parity` → `PASS ...`; `gh run list -R
  RandomGenericUsername/config-assembler-engine --limit 1 --json status,conclusion`.
- [x] Annotated tag `v0.1.0` exists on the released commit, is anonymously fetchable, and its
  GitHub Release carries the wheel + sdist.
  Verify: `GIT_TERMINAL_PROMPT=0 git -c credential.helper= ls-remote
  https://github.com/RandomGenericUsername/config-assembler-engine 'refs/tags/v0.1.0'` → one line;
  `gh release view v0.1.0 --json assets --jq '.assets[].name'` → both artifacts.
- [x] `git status --porcelain` is empty in both repos; `dotfiles-v3` `master` is at the same SHA
  as before the slice.
  Verify: `git status --porcelain` → empty; `git -C /home/inumaki/Development/dotfiles-v3 rev-parse
  master` → `843b50d0d000d3497b625627bc58fb6427366b59`.

## Out of scope

- Rewiring `dotfiles-v3` (`csg` / `weg` / `itr` `pyproject.toml`) to the tag, regenerating their
  locks, deleting `src/shared/config-assembler-engine/`. Those are the remaining S2 steps in that
  repo.
- Any package behaviour change: no strategy, parser, validator, override, merge or error-shape
  change; no dependency change.
- Modernizing the imported test modules for ruff `UP`/`I001` (84 findings, all in `tests/`);
  `make lint` covers `src/` as the in-tree Makefile did.
- PyPI publication; the channel is the git tag plus GitHub Release artifacts.

## Closeout

Closed 2026-10-19 on `ticket/req-001-closeout-close-ticket` (worktree
`.worktrees/req-001-closeout/`), fast-forwarded onto `main`. The slice work is on
`ticket/req-001-repo-birth` (`a2d3ad6` planning, `cad8a91` implementation) and the tag `v0.1.0`
points at `cad8a91`; this closeout only ticks the record, writes this section, and moves
`tickets/INDEX.md` to Done.

**Worktree exception (birth only).** The guard creates a branch from a clean `main`, and this
repo had no scaffold on `main` before the first commits. Those two commits were therefore made on
`ticket/req-001-repo-birth` in the clone, per the S2a instruction, and the guard was installed by
them. This closeout is the first item that uses the guard normally:
`bash scripts/worktree-guard.sh create req-001-closeout close-ticket` →
`worktree=.worktrees/req-001-closeout/`, `branch=ticket/req-001-closeout-close-ticket`, exit 0.
Every later item uses it.

Evidence (commands run from the repo root unless a path is named):

| Claim | Command | Result |
| --- | --- | --- |
| Repo was absent before the slice | `gh repo view RandomGenericUsername/config-assembler-engine` | `GraphQL: Could not resolve to a Repository with the name '…'`, exit 1 |
| Repo is public | `gh api repos/RandomGenericUsername/config-assembler-engine --jq .private` | `false`, exit 0 |
| History preserved by subtree split | `git -C /home/inumaki/Development/dotfiles-v3 subtree split -P src/shared/config-assembler-engine -b split/config-assembler-engine` | exit 0; tip `a6aa122714e6e6b9896b34ba5b00183963b3260c`; `git rev-list --count` → `6` |
| `main` starts at the split | `git log --oneline \| tail -1` | `324c282 feat: implement config-assembler-engine with full 7-phase pipeline` |
| Harness byte-identical | `diff -r /home/inumaki/Development/mu/<path> <path>` for `scripts/worktree-guard.sh`, `skills-lock.json`, `_bmad`, `.agents/skills`, `.opencode`, `tickets/_template`, `tickets/_fixtures`, `tickets/LIFECYCLE.md` | empty output, exit 0 for all eight |
| Guard is executable and clean-tree safe | `bash scripts/worktree-guard.sh check-clean` | exit 0 |
| Guard creates a worktree/branch | `bash scripts/worktree-guard.sh create req-001-closeout close-ticket` | `worktree=.worktrees/req-001-closeout/`, `branch=ticket/req-001-closeout-close-ticket`, exit 0 |
| AGENTS carries the contract | `rg -c "Evidence:\|Verify:" AGENTS.md`; `test -L CLAUDE.md` | `8`; symlink, exit 0 |
| Tests (local) | `uv run pytest -q` | `163 passed in 0.24s`, exit 0 |
| Gates (local clone) | `make gates` | test `163 passed`, lint `All checks passed!`, hermetic `163 passed`, `uv lock --check` resolved 16 packages, docs-check silent → exit 0 |
| Build | `make build` | `dist/config_assembler_engine-0.1.0-py3-none-any.whl` + `dist/config_assembler_engine-0.1.0.tar.gz`, exit 0 |
| Specs valid | `openspec validate --specs --strict` | `✓ spec/config-resolution-chain`, `Totals: 1 passed, 0 failed (1 items)`, exit 0 |
| Change valid | `openspec validate repo-birth --strict` | `Change 'repo-birth' is valid`, exit 0 |
| Engine is tool-agnostic | `rg -ni "colorscheme\|wallpaper\|icon_renderer" src` | exit 1 (no matches) |
| Workflow ↔ Makefile parity | `make ci-gates-parity` | `PASS gates=['docs-check', 'hermetic', 'lint', 'lock-check', 'test'] jobs=['ci-gates-parity', 'docs-check', 'hermetic', 'lint', 'lock-check', 'release', 'test'] exempt=['ci-gates-parity', 'release']`, exit 0 |
| Fresh clone at HEAD is green | `git clone <url> /tmp/opencode/cae-fresh && make sync && make gates && make build` | exit 0 for all three; `163 passed` twice, `All checks passed!`, both artifacts in `dist/` (HEAD `cad8a91c6bcbc4087209911bc36d82f8b6349f66`) |
| First `main` CI run is green | `gh run view 39355235533 --json status,conclusion,jobs` | `success`; jobs `test`, `lint`, `hermetic`, `lock-check`, `docs-check`, `ci-gates-parity` all `success`; `release` `skipped` (not a tag) — https://github.com/RandomGenericUsername/config-assembler-engine/actions/runs/39355235533 |
| Tag CI run is green including release | `gh run view 39355306264 --json status,conclusion,jobs` | `success`; all six gate jobs `success` and `release` `success` — https://github.com/RandomGenericUsername/config-assembler-engine/actions/runs/39355306264 |
| Tag is annotated and points at the tested commit | `git rev-parse v0.1.0`; `git rev-parse v0.1.0^{commit}`; `git cat-file -t v0.1.0` | tag `5c5cc84e4103f1135a717320cf7cb809487d6e5e`; commit `cad8a91c6bcbc4087209911bc36d82f8b6349f66`; `tag` |
| Tag is anonymously fetchable, one line | `GIT_TERMINAL_PROMPT=0 git -c credential.helper= ls-remote https://github.com/RandomGenericUsername/config-assembler-engine 'refs/tags/v0.1.0'` | `5c5cc84e4103f1135a717320cf7cb809487d6e5e	refs/tags/v0.1.0` — exactly one line, exit 0 |
| Release carries both artifacts | `gh release view v0.1.0 --json assets --jq '.assets[].name'` | `config_assembler_engine-0.1.0-py3-none-any.whl`, `config_assembler_engine-0.1.0.tar.gz`; `isDraft: false` |
| `dotfiles-v3` untouched | `git -C /home/inumaki/Development/dotfiles-v3 rev-parse master`; `git status --porcelain \| wc -l` | `843b50d0d000d3497b625627bc58fb6427366b59`; `0` |

Note: `skills-lock.json`'s `computedHash` values do not equal a plain sha256 of `SKILL.md` — the
same is true in `mu` (verified by running the same recomputation in both repos). The requirement
is that the harness is copied, so the evidence is byte identity plus the lock file being
byte-identical, not a re-derivation of `bmod`'s hash scheme.

Slice commits (oldest first, all on `main`):

| Commit | Subject |
| --- | --- |
| `a2d3ad6` | `plan(REQ-001): open repo-birth ticket, OpenSpec change, and release docs` |
| `cad8a91` | `feat(REQ-001): mu-strict harness, fresh-clone gates, and release CI` |

## Closeout 2 — re-verification and archive (2026-10-21)

The record above was written on `2026-10-19` when the change was still open. This second pass,
on `ticket/req-001-archive-archive-change` (worktree `.worktrees/req-001-archive/`, created by
`bash scripts/worktree-guard.sh create req-001-archive archive-change` → exit 0), **re-ran every
gate before trusting any ticked box**, then archived the OpenSpec change so the ticket's last open
link closes.

Re-verification results (all commands from the repo root at `main` `d72a10b`):

| Claim | Command | Result |
| --- | --- | --- |
| Tree clean before ticking | `git status --porcelain`; `bash scripts/worktree-guard.sh check-clean` | empty output; exit 0, exit 0 |
| Gate set is green now | `make gates` | `163 passed` (test), `All checks passed!` (lint), `163 passed` (hermetic), `Resolved 16 packages` (lock-check), docs-check silent → exit 0 |
| Workflow ↔ Makefile parity | `make ci-gates-parity` | `PASS gates=['docs-check', 'hermetic', 'lint', 'lock-check', 'test'] jobs=['ci-gates-parity', 'docs-check', 'hermetic', 'lint', 'lock-check', 'release', 'test'] exempt=['ci-gates-parity', 'release']`, exit 0 |
| Packaging builds from a clean `dist/` | `rm -rf dist && uv build` | `Successfully built dist/config_assembler_engine-0.1.0.tar.gz` + `dist/config_assembler_engine-0.1.0-py3-none-any.whl`, exit 0 |
| The published tag installs and imports in isolation | `uv venv /tmp/opencode/cae-tag-venv && uv pip install --no-cache "git+…@v0.1.0"` then `python -c 'importlib.metadata.version + import'` | install exit 0; `version: 0.1.0`; `missing_public_names: []`; `direct_url` records `requested_revision: v0.1.0`, `commit_id: cad8a91c6bcbc4087209911bc36d82f8b6349f66` |
| Tag still objects as stated | `git rev-parse v0.1.0`; `git rev-parse 'v0.1.0^{commit}'`; `git cat-file -t v0.1.0` | `5c5cc84e4103f1135a717320cf7cb809487d6e5e`; `cad8a91c6bcbc4087209911bc36d82f8b6349f66`; `tag` |
| Release still carries both artifacts | `gh release view v0.1.0 --json assets` | `{"assets":["config_assembler_engine-0.1.0-py3-none-any.whl","config_assembler_engine-0.1.0.tar.gz"],"draft":false,"prerelease":false,"tag":"v0.1.0"}`, exit 0 |
| Last three CI runs are green | `gh run list -R RandomGenericUsername/config-assembler-engine --limit 3 --json databaseId,conclusion` | `39355451177` / `39355306264` / `39355235533`, all `conclusion: success` |
| Change archived, specs valid | `openspec archive repo-birth --yes`; `openspec validate --all --strict` | `Change 'repo-birth' archived as '2026-10-21-repo-birth'.`; `✓ spec/config-resolution-chain`, `✓ spec/repo-harness`, `Totals: 2 passed, 0 failed`, exit 0 |

**`uv tool install` does not apply to this package.** The prescribed isolated check,
`uv tool install "git+…@v0.1.0"`, exits `2` with
`No executables are provided by package 'config-assembler-engine'; removing tool` /
`error: Failed to install entrypoints` — correct behaviour for a library that exposes no console
script, not a release defect. The equivalent isolated proof is a throwaway `uv venv` plus
`uv pip install` of the same tag (row above): version `0.1.0`, all twelve public names importable,
and `direct_url.json` pinning the peeled commit. Reproduce in one command:
`uv venv /tmp/x && uv pip install --python /tmp/x/bin/python "git+https://github.com/RandomGenericUsername/config-assembler-engine@v0.1.0" && /tmp/x/bin/python -c 'import importlib.metadata as md; print(md.version("config-assembler-engine"))'` → `0.1.0`.

**Archive note.** Archiving moved the change to `openspec/changes/archive/2026-10-21-repo-birth/`
and merged its delta into the new `openspec/specs/repo-harness/spec.md`. The acceptance check
`openspec validate repo-birth --strict` was accurate while the change was active; once archived
the equivalent checks are `openspec validate --specs --strict` and `openspec validate --all
--strict`, both `Totals: 2 passed, 0 failed`, exit 0. Archiving wrote a placeholder `## Purpose`
into `repo-harness`, which `--strict` rejected; it was replaced with a real purpose paragraph in
this same commit before validation was re-run.
