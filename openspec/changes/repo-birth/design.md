# Design — repo birth

## Decisions

### D1. History comes from `git subtree split`, not a fresh root commit

The in-tree package has six commits, including the S0b hermetic-fixture commit and the CSG
directory-resolution commit. A fresh import would make the extracted repo's first commit unrelated
to the code's history and would lose the "why" of both. `git subtree split -P
src/shared/config-assembler-engine` writes a branch whose tree is the package directory and whose
commits are the six that touched it.

Evidence: `git -C /home/inumaki/Development/dotfiles-v3 subtree split -P
src/shared/config-assembler-engine -b split/config-assembler-engine` → exit 0, tip `a6aa122`,
`git rev-list --count` → `6`.
Verify: `git log --oneline` in this repo → six commits, oldest `feat: implement
config-assembler-engine with full 7-phase pipeline`.

Consequence: the split branch was pushed as `main` in one push, so `main` starts at the imported
tree, and the scaffold lands one merge later.

### D2. Harness files are copied, not re-authored

AD-12 requires the same `worktree-guard.sh` and the same `skills-lock.json` hashes as `mu`. The
copy set is exactly: `scripts/worktree-guard.sh`, `skills-lock.json`, `_bmad/`, `.agents/skills/`,
`.opencode/`, `tickets/_template/`, `tickets/_fixtures/`, `tickets/LIFECYCLE.md`.
`tickets/INDEX.md` is repo-specific (an empty ledger needs the counting rule, not `mu`'s 100+
rows), and `AGENTS.md` is retargeted but keeps the evidence rule verbatim.

Evidence: `diff` per path against `mu` is empty (see `REQ-001-repo-birth`'s evidence table).
Verify: `diff -r /home/inumaki/Development/mu/_bmad _bmad && diff /home/inumaki/Development/mu/skills-lock.json skills-lock.json`.

### D3. One branch exception, then the guard

The guard refuses to create a worktree from a dirty tree and this repo has no `main` history to
branch from before the first scaffold commit. The first commits are therefore made on
`ticket/req-001-repo-birth` in the clone (the slice instruction), and every later item uses
`bash scripts/worktree-guard.sh create <id> <slug>`.

Evidence: `scripts/worktree-guard.sh` is copied and executable in the implementation commit; the
ticket records `worktree: none` and the exception in its Closeout.
Verify: `bash scripts/worktree-guard.sh check-clean` → exit 0 on a clean tree.

### D4. The Makefile's `GATES` is the single gate list; CI is compared to it

The workflow has one job per gate so a failure names the gate, but a hand-maintained job list
drifts. `scripts/ci-gates-parity.py` parses `GATES` from the `Makefile` and the job keys from the
workflow and fails on any unmatched entry in either direction. CI-only jobs (`ci-gates-parity`,
`release`) are declared with a reason; a stale declaration fails.

Evidence: `make ci-gates-parity` → `PASS gates=[...] jobs=[...] exempt=[...]`.
Verify: `make ci-gates-parity`.

### D5. Hermetic means hermetic, not "green on this host"

AD-8 is already implemented in the imported tests (`tests/conftest.py` +
`tests/test_hermetic_canary.py`). The Makefile adds `hermetic`, which runs the suite with `HOME`
and all four `XDG_*` roots pointed at fresh temp dirs; the fixture does the `chdir`. `make gates`
includes it, so a host-dependent test cannot pass a release gate.

Evidence: the canary asserts every `XDG_*` value is inside the pytest tmp root and that the
host-only `DOTFILES_*` / `*_CONFIG_FILE_PATH` variables are absent.
Verify: `make hermetic` → exit 0, same count as `make test`.

### D6. `lint` covers `src/` and the source was made clean mechanically

The in-tree Makefile linted `src/` only, and the imported source was not clean (21 findings: 9
fixable import-order/unused-import issues, 8 line-length, `NotADirectoryError_` naming). The birth
commit fixes only the mechanical class — `ruff check --fix src` + `ruff format src` — and keeps the
public exception name via a `N801`/`N818` per-file ignore, because renaming an exported class is a
breaking change, not a lint fix. Tests are not linted: the imported modules carry 84 pre-existing
`UP`/`I001` findings whose fix would rewrite test code, which a birth slice must not do.

Evidence: `uv run ruff check src` → `All checks passed!`; `uv run pytest -q` → `163 passed`
(identical to the pre-fix count).
Verify: `make lint && make test`.

## Verification plan

| Decision | Command | Expected |
| --- | --- | --- |
| D1 | `git log --oneline \| wc -l` | six imported commits before the scaffold |
| D2 | `diff -r <mu-path> <path>` for each copied path | empty output, exit 0 |
| D3 | `bash scripts/worktree-guard.sh check-clean` | exit 0 |
| D4 | `make ci-gates-parity` | `PASS ...` exit 0 |
| D5 | `make hermetic` | exit 0, same test count as `make test` |
| D6 | `make lint` | exit 0 |
| Packaging | `uv lock --check && uv build` | exit 0; `dist/config_assembler_engine-0.1.0-*.whl` + `.tar.gz` |
| Release | `git ls-remote --tags origin v0.1.0` | exactly one line (tag object) + peeled commit |
