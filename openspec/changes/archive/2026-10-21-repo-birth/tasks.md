# Tasks

## 1. Import and scaffold (≤2h)

- [x] 1.1 Create the public repo and push the subtree-split branch as `main`; verify the six
  imported commits are present.
  Verify: `git log --oneline | wc -l` → `6` before the scaffold commit; `gh api
  repos/RandomGenericUsername/config-assembler-engine --jq .private` → `false`.
- [x] 1.2 Copy the mu harness byte-for-byte (`scripts/worktree-guard.sh`, `skills-lock.json`,
  `_bmad/`, `.agents/skills/`, `.opencode/`, `tickets/_template/`, `tickets/_fixtures/`,
  `tickets/LIFECYCLE.md`).
  Verify: `diff` against `/home/inumaki/Development/mu/<path>` for each → empty, exit 0; `test -x
  scripts/worktree-guard.sh` → exit 0.
- [x] 1.3 Write `AGENTS.md` (evidence rule, config-resolution charter, ticket routing, worktree
  discipline, planning-vs-implementation, release ritual) and `CLAUDE.md` → `AGENTS.md`.
  Verify: `test -L CLAUDE.md` → exit 0; `rg -c "Evidence:|Verify:" AGENTS.md` → ≥ 2.

## 2. Ledger and specs (≤2h)

- [x] 2.1 Add `tickets/INDEX.md` with the counting-rule header and `tickets/REQ-001-repo-birth/`.
  Verify: `rg -n "REQ-001" tickets/INDEX.md tickets/REQ-001-repo-birth/ticket.md` → both hit.
- [x] 2.2 Add `openspec/config.yaml`, `openspec/specs/config-resolution-chain/spec.md`, and the
  `repo-birth` change (proposal, design, tasks, spec delta).
  Verify: `openspec validate --specs --strict` and `openspec validate repo-birth --strict` → exit 0.

## 3. Gates and release (≤2h)

- [x] 3.1 Add the fresh-clone `Makefile` and `scripts/ci-gates-parity.py`; make every verb run
  from `$(CURDIR)`.
  Verify: `make gates` → exit 0; `make ci-gates-parity` → `PASS ...`.
- [x] 3.2 Add `.github/workflows/gates.yml` with one job per gate, `ci-gates-parity`, and a
  tag-only `release` job.
  Verify: `make ci-gates-parity` → exit 0 (the workflow is parsed, not grepped).
- [x] 3.3 Make the imported tree lint-clean with mechanical fixes only, then prove packaging on a
  fresh clone: `uv sync --frozen`, `uv run pytest -q`, `uv build`.
  Verify: `make lint` → exit 0; `uv run pytest -q` → `163 passed`; `ls dist/config_assembler_engine-0.1.0-*`
  lists a wheel and an sdist.
- [x] 3.4 Tag annotated `v0.1.0` after the gates and CI exist; push the tag; confirm the Release
  carries both artifacts.
  Verify: `GIT_TERMINAL_PROMPT=0 git -c credential.helper= ls-remote https://github.com/RandomGenericUsername/config-assembler-engine 'refs/tags/v0.1.0'`
  → exactly one line; `gh release view v0.1.0 --json assets --jq '.assets[].name'` → wheel + sdist.

## 4. Closeout (≤0.5h)

- [x] 4.1 Tick acceptance, record commands + results in the ticket, move the INDEX entry to Done.
  Verify: `git status --porcelain` → empty after the merge; `rg -n "REQ-001" tickets/INDEX.md` →
  the Done section lists it.
