# config-assembler-engine

A reusable, stateless, hexagonal configuration resolution engine for Python >= 3.12.

Give it a **policy** (where to look), a **schema** (what the settings look like) and **override
rules** (which fields env vars / CLI args may change). It finds the settings file, parses it,
validates it, applies the allowlisted overrides, and returns a validated model plus the resolved
path and every override it applied.

The engine is deliberately tool-agnostic: it does not know any tool's XDG subdirectory or settings
filename. The consumer passes its own strategy chain, filename and env prefix.

## Public surface

```python
from config_assembler_engine import (
    AppliedOverride, AssembleConfiguration, AssembleDir, AssemblyResult,
    ConfigAssemblerError, DirAssemblyResult, NotADirectoryError_,
    OverrideRule, OverrideSource, PathSource, ResolutionPolicy,
    ResourceKind, ResolvedPath,
)
```

Adapters live below the root export and are imported from their modules:

- `adapters.factories` — `create_standard_assembler(parser, strategies=…)`,
  `create_directory_assembler(strategies=…, file_pattern=…)`
- `adapters.strategies` — `CliPathStrategy`, `EnvPathStrategy`, `DirectoryTraversalStrategy`,
  `XdgStrategy`, `DefaultFileStrategy` (plus `*DirStrategy` variants for directory resources)
- `adapters.parsers` — `TomlConfigParser`, `YamlConfigParser`, `JsonConfigParser`
- `adapters.config_validator` / `adapters.env_reader` / `adapters.type_coercer` — the default
  Pydantic validator, `os.environ` reader and type coercer

## The resolution chain

`CompositePathResolver` runs the strategy list **in the order you give it** and returns the first
hit; if nothing hits it raises `PathResolutionError`. The chain the CLIs use, highest priority
first:

| # | Strategy | Source reported | Looks at |
| --- | --- | --- | --- |
| 1 | `CliPathStrategy` | `PathSource.CLI_PATH` | the explicit `--config` path handed to `execute(explicit_path=…)` |
| 2 | `EnvPathStrategy` | `PathSource.ENV_PATH` | `$<PREFIX>_CONFIG_FILE_PATH` |
| 3 | `DirectoryTraversalStrategy` | `PathSource.DIRECTORY` | the filename in `cwd`, then in each parent up to `max_levels` |
| 4 | `XdgStrategy` | `PathSource.XDG` | `$XDG_CONFIG_HOME/<subdir>/<filename>` (falls back to `~/.config`) |
| 5 | `DefaultFileStrategy` | `PathSource.DEFAULT` | the packaged/default path given by the consumer |

Evidence: `src/config_assembler_engine/adapters/path_resolver.py:6-21` and one file per strategy
under `src/config_assembler_engine/adapters/strategies/`.

Every strategy checks existence (`is_file` for `ResourceKind.FILE`, `is_dir` for
`ResourceKind.DIRECTORY`) and returns `None` when the candidate is absent, so a miss falls through
to the next strategy instead of failing.

## The pipeline

`AssembleConfiguration.execute(policy, rules, schema, cli_overrides=None, explicit_path=None)`:

1. resolve the path (chain above)
2. parse it with the injected parser
3. validate the parsed dict against the Pydantic `schema`
4. read env variables matching `<PREFIX>__<KEY>` (double underscore)
5. match env + CLI overrides against the registered `OverrideRule`s
6. coerce each string to the field's type and merge it (`ConfigMergeService`)
7. re-validate the merged dict — a failure carries `applied_overrides` for diagnosis

Overrides are applied env-first, CLI-last, so the CLI wins for the same field. Only fields named
in an `OverrideRule` can be overridden.

`AssembleDir.execute(policy, explicit_path=None)` runs the same chain for a directory resource and
returns `DirAssemblyResult(directory, source, files)` with the matched files sorted.

## Quickstart

```python
from pathlib import Path

from pydantic import BaseModel, Field

from config_assembler_engine import OverrideRule, ResolutionPolicy, ResourceKind
from config_assembler_engine.adapters.factories import create_standard_assembler
from config_assembler_engine.adapters.parsers import YamlConfigParser
from config_assembler_engine.adapters.strategies import (
    CliPathStrategy, DefaultFileStrategy, DirectoryTraversalStrategy, EnvPathStrategy, XdgStrategy,
)


class AppConfig(BaseModel):
    engine: str = Field(default="docker")
    timeout: int = Field(default=30)


strategies = [
    CliPathStrategy(kind=ResourceKind.FILE),
    EnvPathStrategy(var="CONFIG_FILE_PATH", kind=ResourceKind.FILE),
    DirectoryTraversalStrategy(filename="config.yaml", max_levels=2, kind=ResourceKind.FILE),
    XdgStrategy(xdg_subdir="my-app", filename="config.yaml", kind=ResourceKind.FILE),
    DefaultFileStrategy(path=Path("defaults.yaml"), kind=ResourceKind.FILE),
]

assembler = create_standard_assembler(parser=YamlConfigParser(), strategies=strategies)
result = assembler.execute(
    policy=ResolutionPolicy(env_prefix="MY_APP"),
    rules=[OverrideRule(field_path="timeout", sources={"env", "cli"})],
    schema=AppConfig,
    explicit_path=None,
)
print(result.config, result.resolved_path.source, result.applied_overrides)
```

## Install

From a checkout (development):

```bash
make sync          # uv sync --frozen
make gates         # test + lint + hermetic + lock-check + docs-check
```

As a consumer, pin an immutable tag — never a path and never a branch:

```toml
# pyproject.toml
dependencies = ["config-assembler-engine>=0.1.0"]

[tool.uv.sources]
config-assembler-engine = { git = "https://github.com/RandomGenericUsername/config-assembler-engine", tag = "v0.1.0" }
```

```bash
uv add "config-assembler-engine @ git+https://github.com/RandomGenericUsername/config-assembler-engine@v0.1.0"
```

A published tag is never moved or deleted; a fix ships as a new tag.

## What `dotfiles-v3` pins

`dotfiles-v3` consumes this repo as a pinned artifact (`v0.1.0` at first release) from the three
CLIs only:

- `src/cli-tools/color-scheme-generator/pyproject.toml` (`csg`)
- `src/cli-tools/wallpaper-effects-generator/pyproject.toml` (`weg`)
- `src/cli-tools/icon-templates-renderer/pyproject.toml` (`itr`)

The runtime and provisioning do not depend on this package (spine AD-1). No consumer may use
`path = ...` or `editable = true`.

## Design record

- `docs/SPEC.md` — the specification the package was implemented against.
- `docs/ADR.md` — the accepted decisions (strategy order, override precedence, error shapes).
- `docs/PROTOTYPE.md` — the prototype plan and the TDD status at extraction time.

## Release

Version in `pyproject.toml` + `CHANGELOG.md` entry → annotated tag `vX.Y.Z` → GitHub Actions
builds the sdist and wheel and attaches them to the GitHub Release. PyPI is not used.

## Repo harness

- `AGENTS.md` (with `CLAUDE.md` symlinked to it) is the operating contract: evidence rule,
  worktree discipline, planning-vs-implementation commits.
- `tickets/` is the work ledger (`INDEX.md`, `LIFECYCLE.md`, `_template/`).
- `openspec/` holds the specs and the change log (`config.yaml`, `specs/`, `changes/`).
- `scripts/worktree-guard.sh create <id> <slug>` makes `.worktrees/<id>/` on
  `ticket/<id>-<slug>`; never work on `main`.
- `.agents/skills/` + `skills-lock.json` and `_bmad/` are hash-pinned copies of the `mu` harness.

## License

MIT — see `LICENSE`.
