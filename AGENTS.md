# AGENTS.md — config-assembler-engine operating contract

Canonical agent instructions for this repo. Claude (`CLAUDE.md` symlinks here), OpenCode, and all
BMAD/OpenSpec skills must follow this.

## 1. Evidence rule (hard requirement)

No suggestion "in the air". Every technical claim, recommendation, or design decision must carry testable evidence — at least one of:

- **Doc evidence:** file path + line range, or commit SHA, or URL + quoted section.
- **Prototype evidence:** a script/command in-repo that runs and prints the validating output. State the exact command and observed result.
- **Test evidence:** `command run → result` (exit code + key output). Failing-as-expected counts if it validates the claim.

Format: any non-trivial claim gets an `Evidence:` line and a `Verify:` line (how to reproduce in one command). If evidence doesn't exist yet, say `Evidence: MISSING — run <X> to produce it` and do not present the claim as fact.

## 2. Charter — the config resolution engine

- This repo owns one package: `config-assembler-engine` (import package
  `config_assembler_engine`). It resolves *where* a tool's settings live and *what* the settings
  are: a strategy chain (CLI path → env path → cwd traversal → XDG → default file), a parser
  port (TOML/YAML/JSON adapters), a Pydantic validator, an env/CLI override matcher, a type
  coercer, and a merge that ends in a re-validation.
- Public surface: `AssembleConfiguration`, `AssembleDir`, `ResolutionPolicy`, `OverrideRule`,
  `OverrideSource`, `ResourceKind`, `PathSource`, `ResolvedPath`, `AssemblyResult`,
  `DirAssemblyResult`, `AppliedOverride`, `ConfigAssemblerError`, `NotADirectoryError_`.
  Evidence: `src/config_assembler_engine/__init__.py:15-29`.
- Runtime dependencies are `pydantic` and `pyyaml` only. The package imports no sibling repo and
  reads no path in another repo.
  Evidence: `pyproject.toml` `dependencies = ["pydantic>=2.0", "pyyaml>=6.0.3"]`.
- **It does not know any tool's XDG subdir or settings filename.** Each consumer supplies its own
  strategies, filename and env prefix; the engine only runs the ordered chain it is handed.
  Evidence: `src/config_assembler_engine/adapters/path_resolver.py:6-21`;
  `src/config_assembler_engine/adapters/strategies/xdg.py:8-12`.
- Consumers are the three CLIs only — `csg`, `weg`, `itr` (spine AD-1 keeps the runtime out of
  every CLI-side package; provisioning does not depend on this one either).
  Evidence: `rg -n 'config-assembler-engine' src --glob pyproject.toml` in `dotfiles-v3` →
  `src/cli-tools/{color-scheme-generator,wallpaper-effects-generator,icon-templates-renderer}/pyproject.toml`.
- Consumers pin an immutable git tag, never a path and never `editable = true` (spine AD-3).

## 3. How we work here

- Read files, run commands (`rg`, `git log`, `uv`, `pytest`, `openspec validate`), paste results.
  Prefer executed output over recalled text.
- When findings contradict an earlier claim, state the discrepancy and trust the evidence.
- Keep changes small, verifiable, committable. A suite that is green only on the author's host is
  not green: tests are hermetic (empty `HOME` / `XDG_*`, foreign cwd), enforced by
  `tests/conftest.py` plus the canary in `tests/test_hermetic_canary.py`.
  Verify: `make hermetic` → exit 0.
- `make lint` covers `src/` only, matching the in-tree Makefile this package was extracted with.
  The imported test modules carry pre-existing `UP`/`I001` findings; modernizing them is a
  separate ticket, not part of a birth slice.

## 4. OpenSpec + BMAD routing (via tickets)

- All work starts as a ticket in `tickets/` (`_template/`, vocab + rollup in `INDEX.md`, flow in
  `LIFECYCLE.md`). Triage with `/triage` (lanes in `tickets/_fixtures/router-lanes.md`).
- Broad scope (package charter, public API evolution, dependency policy) → BMAD. Installed
  skills are exactly the seven in `skills-lock.json`: `bmad`, `bmad-architecture`, `bmad-build`,
  `bmad-product-brief`, `bmad-spec`, `bmod-core-tools`, `bmod-method` — say so when a step needs a
  missing skill instead of improvising.
- Bounded slices (one proposal + spec + design + tasks) → OpenSpec
  (`/opsx-propose`, `/opsx-apply`, `/opsx-archive`). Artifact rules in `openspec/config.yaml`
  restate this evidence rule per artifact.
- Never work on main: `scripts/worktree-guard.sh create <id> <slug>` scaffolds
  `ticket/<id>-<slug>` + `.worktrees/<id>/`, and aborts when the tree is dirty. Record
  worktree/branch in the ticket.

## 5. Planning vs implementation

- Planning edits (tickets, `openspec/` changes, docs) and implementation changes (code, packaging,
  harness) land in **separate commits**.
- Never mix planning edits with implementation in one commit or one turn.

## 6. Release ritual

- Green gates on the ticket branch → version bumped in `pyproject.toml` + `CHANGELOG.md` →
  annotated tag `vMAJOR.MINOR.PATCH` → CI builds sdist + wheel and attaches them to the GitHub
  Release → consumer pin bump in its own commit.
- A published tag is never moved or deleted; a fix is a new tag. PyPI is not a channel.
  Evidence: `CHANGELOG.md` `0.1.0`; `git tag -l`.

## 7. Gates

- `make gates` = `test lint hermetic lock-check docs-check`; `make build` produces the sdist and
  wheel. CI (`.github/workflows/gates.yml`) runs one job per gate target plus `ci-gates-parity`,
  which compares the workflow's job set against `GATES` in the `Makefile`.
  Verify: `make gates && make build` → exit 0; `make ci-gates-parity` → exit 0.
