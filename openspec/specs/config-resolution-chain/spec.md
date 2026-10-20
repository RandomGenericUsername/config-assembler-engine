# config-resolution-chain Specification

## Purpose

`config-assembler-engine` resolves *where* a tool's settings live and *what* the settings are.
This spec fixes the resolution chain, the override semantics, the result/error shapes, and the
boundary that keeps the engine tool-agnostic (spine AD-1, AD-2).

The chain the consumers build, highest priority first:

| # | Strategy | `PathSource` | Candidate |
| --- | --- | --- | --- |
| 1 | `CliPathStrategy` | `CLI_PATH` | `explicit_path` passed to `execute` |
| 2 | `EnvPathStrategy` | `ENV_PATH` | `$<env_prefix>_CONFIG_FILE_PATH` |
| 3 | `DirectoryTraversalStrategy` | `DIRECTORY` | the filename in `cwd`, then in parents up to `max_levels` |
| 4 | `XdgStrategy` | `XDG` | `$XDG_CONFIG_HOME/<xdg_subdir>/<filename>`, falling back to `~/.config` |
| 5 | `DefaultFileStrategy` | `DEFAULT` | the consumer-supplied default path |

Source: one file per strategy under `src/config_assembler_engine/adapters/strategies/`.

## Requirements

### Requirement: resolution is an ordered, first-hit strategy chain

`CompositePathResolver.resolve(policy, explicit_path)` SHALL run the injected strategies in the
order given, return the first non-`None` result, and raise `PathResolutionError` when every
strategy misses. The engine SHALL NOT reorder, sort or deduplicate the chain.

Source: `src/config_assembler_engine/adapters/path_resolver.py:6-19`.

#### Scenario: the first hit wins

- **WHEN** two strategies both match (e.g. an explicit path and an XDG file)
- **THEN** the result is the earlier strategy's `ResolvedPath`, and its `source` names that
  strategy

#### Scenario: an empty chain fails loudly

- **WHEN** no strategy resolves and `resolve` is called
- **THEN** it raises `PathResolutionError("No strategy found a config file")`

### Requirement: a strategy miss falls through, it does not fail

Every strategy SHALL check that its candidate exists (`is_file` for `ResourceKind.FILE`, `is_dir`
for `ResourceKind.DIRECTORY`) and SHALL return `None` when it does not, so a missing candidate
falls through to the next strategy.

Source: `src/config_assembler_engine/adapters/strategies/cli_path.py`,
`.../env_path.py`, `.../directory.py`, `.../xdg.py`, `.../default_file.py`.

#### Scenario: an absent env path is not an error

- **WHEN** `$<PREFIX>_CONFIG_FILE_PATH` points at a path that does not exist
- **THEN** `EnvPathStrategy.resolve(...)` returns `None` and the chain continues

#### Scenario: a directory strategy rejects a file

- **WHEN** a `ResourceKind.DIRECTORY` strategy finds a file at its candidate path
- **THEN** it returns `None` rather than a `ResolvedPath`

### Requirement: the documented consumer chain resolves in a fixed order

The chain described in **Purpose** SHALL resolve in that order, and each hit SHALL report the
`PathSource` named there.

Source: `src/config_assembler_engine/adapters/strategies/xdg.py:25-26`;
`src/config_assembler_engine/adapters/strategies/env_path.py:22`.

#### Scenario: the XDG variable wins over the home fallback

- **WHEN** `XDG_CONFIG_HOME` is set and the subdir/file exists below it
- **THEN** `XdgStrategy` resolves below `XDG_CONFIG_HOME`, not below `~/.config`

#### Scenario: the env path is prefix-scoped

- **WHEN** `ResolutionPolicy(env_prefix="MY_APP")` is used
- **THEN** `EnvPathStrategy` reads `MY_APP_CONFIG_FILE_PATH` (and an unprefixed
  `CONFIG_FILE_PATH` is ignored)

#### Scenario: traversal is bounded

- **WHEN** the file is above `max_levels` parents from `cwd`
- **THEN** `DirectoryTraversalStrategy` returns `None` instead of walking further

### Requirement: overrides are allowlisted by rule and env precedes CLI

`execute` SHALL apply overrides only for fields registered in an `OverrideRule` whose `sources`
contains that `OverrideSource`, SHALL read env values from `<env_prefix>__<KEY>` (key lowercased,
`__` mapped to `.`), and SHALL apply env before CLI so the CLI wins for the same field.

Source: `src/config_assembler_engine/adapters/env_reader.py:8-17`;
`src/config_assembler_engine/domain/services.py:11-48`;
`src/config_assembler_engine/application/use_cases.py:56-66`.

#### Scenario: an unregistered field is immutable

- **WHEN** `<PREFIX>__ENGINE` is set but no `OverrideRule` registers `engine`
- **THEN** the resulting config keeps the file's value and `applied_overrides` is empty

#### Scenario: the CLI wins over the env for the same field

- **WHEN** a rule allows both sources and both provide `timeout`
- **THEN** the CLI value is merged last and `applied_overrides` lists both, CLI last

#### Scenario: a bad coercion names the field

- **WHEN** an override value cannot be coerced to the field's type
- **THEN** `OverrideCoercionError` is raised carrying `field_path`, `raw_value`, `target_type`
  and `reason`

### Requirement: assembly returns the config, the resolved path and every applied override

`execute` SHALL parse, validate, override and re-validate, and SHALL return
`AssemblyResult(config, resolved_path, applied_overrides)`. A re-validation failure SHALL raise
`ConfigValidationError` carrying the applied overrides.

Source: `src/config_assembler_engine/application/use_cases.py:39-99`;
`src/config_assembler_engine/errors.py:20-28`.

#### Scenario: a clean run reports its provenance

- **WHEN** a valid file resolves and one env override applies
- **THEN** the result carries the validated model, the `ResolvedPath` with its `source`, and one
  `AppliedOverride` with `raw_value` + `coerced_value`

#### Scenario: a failed re-validation is diagnosable

- **WHEN** the merged dict violates the schema
- **THEN** `ConfigValidationError` is raised and `exc.applied_overrides` lists what was merged

### Requirement: directory resolution is a separate use case

`AssembleDir.execute` SHALL resolve through the same chain, raise `NotADirectoryError_` when the
resolved path is not a directory, and return `DirAssemblyResult(directory, source, files)` with
the files matched by `file_pattern` and sorted.

Source: `src/config_assembler_engine/application/use_cases.py:102-125`.

#### Scenario: a directory resource returns sorted members

- **WHEN** the resolved path is a directory containing matching files
- **THEN** `files` is sorted and `source` names the strategy that resolved it

#### Scenario: a file where a directory is required fails loudly

- **WHEN** the resolved path is a file
- **THEN** `NotADirectoryError_` is raised naming the path

### Requirement: the engine is tool-agnostic and imports no sibling repo

The package SHALL name no tool's XDG subdirectory, settings filename or env prefix in its source,
SHALL depend on `pydantic` and `pyyaml` only, and SHALL NOT import `cli_output`, `oci_runtime` or
`cli_invocation_contract`.

Source: spine AD-1, AD-2; `pyproject.toml` `[project].dependencies`;
`src/config_assembler_engine/adapters/factories.py:28-57` (the defaults are a generic shape, not a
tool's).

#### Scenario: no sibling import

- **WHEN** `rg -n "^(from|import) (cli_output|oci_runtime|cli_invocation_contract)" src` runs at the
  repo root
- **THEN** it exits 1 (no matches)

#### Scenario: no tool name in the engine

- **WHEN** `rg -ni "colorscheme|wallpaper|icon_renderer" src` runs
- **THEN** it exits 1 (no matches)

### Requirement: one package, repo root is the package root

The repo SHALL hold one `pyproject.toml` at its root and one import package at
`src/config_assembler_engine/`; tests SHALL live in `tests/` and there SHALL be no nested project
root or second OpenSpec root under `src/`.

Source: spine AD-2.

#### Scenario: layout on a fresh clone

- **WHEN** `test -f pyproject.toml -a -d src/config_assembler_engine -a -d tests` runs at the repo
  root
- **THEN** it exits 0, and `find src -name pyproject.toml` prints nothing

#### Scenario: the suite runs from the root

- **WHEN** `make test` runs on a fresh clone
- **THEN** exit 0 and the reported test count covers every module in `tests/`
