# Changelog

Notable changes to the `config-assembler-engine` repo. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versions follow the git tags
(`vMAJOR.MINOR.PATCH`, annotated, never moved).

## [Unreleased]

## [0.1.0] - 2026-10-19

First release. The package content is the in-tree `src/shared/config-assembler-engine` package
(version `0.1.0` there) imported with `git subtree split`, history preserved. No resolver
behaviour changed: the strategy chain, override precedence and error shapes are exactly the
imported ones.

### Added

- `config-assembler-engine` 0.1.0 — the `config_assembler_engine` import package: the
  `AssembleConfiguration` / `AssembleDir` use cases, the strategy chain
  (`CliPathStrategy`, `EnvPathStrategy`, `DirectoryTraversalStrategy`, `XdgStrategy`,
  `DefaultFileStrategy` plus the `*DirStrategy` directory variants), the TOML/YAML/JSON parser
  adapters, the Pydantic validator, the `os.environ` reader, the type coercer and the
  override-matching / config-merge domain services.
- Repo harness copied byte-for-byte from `mu`: `scripts/worktree-guard.sh`, `skills-lock.json`,
  `_bmad/`, `.agents/skills/`, `.opencode/`, `tickets/_template/`, `tickets/_fixtures/`,
  `tickets/LIFECYCLE.md`.
- `Makefile` fresh-clone verbs (`sync`, `test`, `lint`, `format`, `check`, `hermetic`,
  `lock-check`, `docs-check`, `gates`, `build`, `install`, `clean`, `ci-gates-parity`),
  `.github/workflows/gates.yml`, and `scripts/ci-gates-parity.py`.
- MIT `LICENSE`, `README.md`, `AGENTS.md`, `CLAUDE.md` symlink, ticket `REQ-001-repo-birth`, the
  OpenSpec change `repo-birth`, and the `config-resolution-chain` capability spec.

### Changed

- `pyproject.toml`: added release metadata (`readme`, `license`, `license-files`, classifiers),
  `hatchling>=1.27`, and `[tool.hatch.build.targets.sdist] only-include` so the harness trees stay
  out of the sdist. `[tool.uv.sources]` is empty — the package has no sibling dependency.
- `src/`: import order, unused-import removal and `ruff format` only (7 files formatted, 2 unused
  imports dropped), so the new `lint` gate passes on the imported tree. No logic changed; the
  suite still reports 163 passed. `NotADirectoryError_` keeps its public name via a targeted
  `N801`/`N818` per-file ignore.
- `docs/PROTOTYPE.md.tdd-status.md`: the plan pointer and the package-layout line no longer name
  the monorepo path `src/shared/config-assembler-engine/`.
