---
id: REQ-001
title: "Birth the config-assembler-engine repo (mu-strict) and cut v0.1.0"
kind: requirement
status: in-progress
lane: 3
links:
  openspec: repo-birth
  bmad: none
worktree: none
branch: ticket/req-001-repo-birth
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
- [ ] `.github/workflows/gates.yml` exists and its job set equals `GATES` (plus declared CI-only
  jobs); the first run on `main` is green.
  Verify: `make ci-gates-parity` → `PASS ...`; `gh run list -R
  RandomGenericUsername/config-assembler-engine --limit 1 --json status,conclusion`.
- [ ] Annotated tag `v0.1.0` exists on the released commit, is anonymously fetchable, and its
  GitHub Release carries the wheel + sdist.
  Verify: `GIT_TERMINAL_PROMPT=0 git -c credential.helper= ls-remote
  https://github.com/RandomGenericUsername/config-assembler-engine 'refs/tags/v0.1.0'` → one line;
  `gh release view v0.1.0 --json assets --jq '.assets[].name'` → both artifacts.
- [ ] `git status --porcelain` is empty in both repos; `dotfiles-v3` `master` is at the same SHA
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

Pending.

Evidence (commands run from the repo root unless a path is named):

| Claim | Command | Result |
| --- | --- | --- |
| Repo was absent before the slice | `gh repo view RandomGenericUsername/config-assembler-engine` | `GraphQL: Could not resolve to a Repository…`, exit 1 |
| History preserved by subtree split | `git -C /home/inumaki/Development/dotfiles-v3 subtree split -P src/shared/config-assembler-engine -b split/config-assembler-engine` | exit 0, tip `a6aa1227…`, `git rev-list --count` → `6` |
| `main` starts at the split | `git log --oneline \| tail -1` | `324c282 feat: implement config-assembler-engine with full 7-phase pipeline` |

Slice commits (oldest first):

| Commit | Subject |
| --- | --- |
| (pending) | `plan(REQ-001): open repo-birth ticket, OpenSpec change, and release docs` |
| (pending) | `feat(REQ-001): mu-strict harness, fresh-clone gates, and release CI` |
