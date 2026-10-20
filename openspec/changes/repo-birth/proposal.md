# repo-birth

## Why

`config-assembler-engine` is the second package extracted from `dotfiles-v3` in the CLI/split
programme (spine `SLICES.md` § S2). The package already exists and is green in-tree, but a
repository is not a valid extraction target until it is mu-strict from its first commit
(spine **AD-12**: "harness later" is exactly the failure the guide names).

Evidence: the package is 880 source lines over 16 test modules plus the S0b hermetic canary, and
its suite is green in-tree.
Verify: `find src -name '*.py' | xargs wc -l | tail -1` → `880 total`;
`find tests -name 'test_*.py' | wc -l` → `17`; `uv run pytest -q` → `163 passed`.

Evidence: the split must preserve history with `git subtree split`, not a fresh-root import
(spine OPEN-8, `SLICES.md` § Slice template step 2).
Verify: `git -C /home/inumaki/Development/dotfiles-v3 subtree split -P src/shared/config-assembler-engine -b split/config-assembler-engine`
→ exit 0, branch tip `a6aa122`, 6 commits.

Evidence: the harness must be copied from `mu`, not re-authored (AD-12).
Verify: `diff /home/inumaki/Development/mu/skills-lock.json skills-lock.json` → empty; same for
`scripts/worktree-guard.sh`, `_bmad`, `.agents/skills`, `.opencode`, `tickets/_template`,
`tickets/_fixtures`, `tickets/LIFECYCLE.md`.

Evidence: the consumers are the three CLIs only, so this repo has no sibling dependency to pin.
Verify: `rg -n 'config-assembler-engine' src --glob pyproject.toml` in `dotfiles-v3` → the three
`src/cli-tools/*/pyproject.toml` files, no runtime and no provisioning match.

## What Changes

- Create the public repo `RandomGenericUsername/config-assembler-engine`, default branch `main`,
  from the `git subtree split` result of `src/shared/config-assembler-engine` (6 commits
  preserved).
- Install the mu-strict harness **byte-for-byte**: `scripts/worktree-guard.sh`, `skills-lock.json`,
  `_bmad/`, `.agents/skills/`, `.opencode/`, `tickets/_template/`, `tickets/_fixtures/`,
  `tickets/LIFECYCLE.md`.
- Add the repo contract: `AGENTS.md` (evidence rule, charter = config resolution engine, ticket
  routing, never work on main, planning vs implementation, release ritual), `CLAUDE.md` →
  `AGENTS.md`.
- Add the work ledger: `tickets/INDEX.md` (counting rule header), `tickets/REQ-001-repo-birth/`.
- Add the OpenSpec root: `openspec/config.yaml` (evidence rules, repo context),
  `openspec/specs/config-resolution-chain/` (the resolution chain, override semantics and the
  tool-agnostic boundary), and this change.
- Add a fresh-clone Makefile (`sync`, `test`, `lint`, `format`, `check`, `hermetic`, `lock-check`,
  `docs-check`, `gates`, `gates-ci`, `build`, `install`, `clean`, `ci-gates-parity`),
  `.github/workflows/gates.yml` (one job per gate + `ci-gates-parity` + tag release),
  `scripts/ci-gates-parity.py`, MIT `LICENSE`, `README.md`, `CHANGELOG.md`, `.gitignore`.
- Keep packaging honest: `[tool.uv.sources]` empty (no sibling dependency), the committed
  `uv.lock` current, and `uv build` producing the `config_assembler_engine-0.1.0` sdist + wheel.
- Make `lint` true of the imported tree with mechanical fixes only (import order, two unused
  imports, `ruff format`), and keep `NotADirectoryError_`'s public name via a targeted per-file
  ignore. No resolver logic changed: the suite still reports 163 passed.

## Capabilities

- **New Capabilities:** `repo-harness` — the repo contract, ticket ledger, OpenSpec root, gate set
  and release ritual that make this repo a valid extraction target.

- **Modified Capabilities:** none. `config-resolution-chain` is documented as the existing
  behaviour; no strategy, override, parser or validator behaviour changes.

## Non-goals (out of scope)

- **Not** rewiring `dotfiles-v3`: no consumer pin, no deletion of
  `src/shared/config-assembler-engine`, no lock regeneration there. That is the next slice step and
  a different repo/commit set.
- **Not** changing package behaviour: no new strategy, parser, validator, field or dependency; no
  change to the resolution order or override precedence.
- **Not** adding argv/env/artifact contract data: the invocation contract lives in `contracts`
  (AD-4) and the three CLIs pin it there.
- **Not** modernizing the imported test modules for ruff `UP`/`I001`; `lint` covers `src/` exactly
  as the in-tree Makefile did (follow-up ticket material).
- **Not** publishing to PyPI (OPEN-3): the release channel is a git tag plus GitHub Release
  artifacts.

## Impact

- New: `AGENTS.md`, `CLAUDE.md` (symlink), `Makefile`, `LICENSE`, `README.md`, `CHANGELOG.md`,
  `.gitignore`, `.github/workflows/gates.yml`, `scripts/ci-gates-parity.py`, `openspec/`,
  `tickets/`, and the copied harness trees.
- Modified: `pyproject.toml` (readme/license metadata, hatchling floor, sdist `only-include`,
  ruff per-file ignore); `src/` import order + formatting only (7 files formatted, 2 unused
  imports removed, 2 empty-import diffs in `__init__.py`); `docs/PROTOTYPE.md.tdd-status.md`
  (two monorepo paths).
- Unchanged: every resolver, port, adapter and domain behaviour; the dependency set in `uv.lock`;
  and every file in `dotfiles-v3`.
