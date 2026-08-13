# uv-0124-combined

**Pattern:** `uv-0124-combined`
**PM:** uv
**PM version under test:** 0.12.4
**Categories:** version_constraints, lockfile_format
**Generated:** 2026-08-13

## Purpose

This probe exercises BOTH SCA-relevant changes introduced in
uv 0.12.4 in a single project:

1. **version_constraints** — wildcard comparison parsing with
   whitespace. `pyproject.toml` uses `== X.*` style specifiers
   for all direct deps and `">= 3.8"` (space after `>=`) for
   `requires-python`. uv 0.12.4 fixed how the resolver parses
   version constraints that include a space before the wildcard
   token.

2. **lockfile_format** — preservation of wildcard Python version
   exclusion markers (`!= 3.8.*`) in `uv.lock`. The root project
   allows Python `>= 3.8`, but `anyio == 4.*` requires Python
   `>= 3.9`. uv 0.12.4 preserves the exclusion as:
   `marker = "python_full_version != '3.8.*'"` in the root
   package's dependency entry. Prior versions dropped this
   exclusion marker, making the dep appear unconditional.

## Project structure

```
uv-0124-combined-20260813-213114/
├── .python-version          (3.9 — Mend reads this first)
├── pyproject.toml
├── uv.lock
├── src/
│   └── uv_wildcard_probe/
│       └── __init__.py
├── README.md
└── expected-tree.json
```

## Direct dependencies

| Package    | Specifier in pyproject.toml | Resolved version |
|------------|-----------------------------|------------------|
| anyio      | `== 4.*`                    | 4.6.0            |
| click      | `== 8.*`                    | 8.1.8            |
| packaging  | `== 24.*`                   | 24.2             |

## Transitive dependencies

| Package | Version | Via         |
|---------|---------|-------------|
| idna    | 3.10    | anyio       |
| sniffio | 1.3.1   | anyio       |

## uv 0.12.4 features exercised

### version_constraints

`pyproject.toml` declares:

```toml
requires-python = ">= 3.8"
dependencies = [
    "anyio == 4.*",
    "click == 8.*",
    "packaging == 24.*",
]
```

The `== X.*` form and the space in `">= 3.8"` exercise the
whitespace-aware wildcard comparison parser introduced in
uv 0.12.4. The UA resolver must correctly interpret these as
version constraints rather than parse errors.

### lockfile_format

`uv.lock` contains a `!= 3.8.*` exclusion marker on `anyio`
in the root package's dependency list:

```toml
[[package]]
name = "uv-wildcard-probe"
...
dependencies = [
    { name = "anyio", specifier = "== 4.*",
      marker = "python_full_version != '3.8.*'" },
    ...
]
```

The `resolution-markers` header records both resolution
branches (Python 3.8.* and >=3.9). uv 0.12.4 preserves the
`!= 3.8.*` wildcard form verbatim; prior versions dropped it,
causing over-reporting of `anyio` as unconditional.

## Mend detection expectations

### What Mend MUST detect

- `anyio 4.6.0` with `marker = "python_full_version != '3.8.*'"`,
  NOT as unconditional.
- `click 8.1.8` unconditional.
- `packaging 24.2` unconditional.
- `idna 3.10` and `sniffio 1.3.1` as transitive deps of anyio.

### Mend failure modes (regressions this probe catches)

1. **Specifier parse failure** — `== 4.*` in pyproject.toml
   causes a parse error in the UA specifier parser. `anyio`
   is dropped from the resolved tree. Visible as missing
   `anyio` entry in Mend output.

2. **Wildcard marker stripped** — the `!= 3.8.*` marker in
   `uv.lock` is not recognized and dropped. `anyio` appears
   as an unconditional dep (no marker). Visible as `marker:
   null` when it should be `"python_full_version != '3.8.*'"`.

3. **Marker parse failure** — the `!= '3.8.*'` expression in
   the marker causes a parse exception and `anyio` is dropped
   entirely. Visible as missing `anyio` in Mend output.

## Python version detection

Mend reads Python version using the PIP precedence chain:
1. `.python-version` file (highest precedence — **this probe**
   uses `3.9`)
2. `pyproject.toml` `[project] requires-python` (`>= 3.8`)

Mend will use `.python-version` (value: `3.9`). The
`requires-python = ">= 3.8"` in `pyproject.toml` is wider
and is relevant for understanding the lock's resolution
branches, but `.python-version` drives the scanner's
Python selection.

## Mend config

**Bucket B — conditional emit.** This probe targets a
version-specific behavior in uv 0.12.4, so a `.whitesource`
is emitted to pin the Python version and document the
version-variation context.

```json
{
  "scanSettings": {
    "versioning": {
      "python": "3.9.21"
    }
  }
}
```

Note: the `uv` tool itself is NOT in the install-tool list
and cannot be pinned via `versioning`. The uv version is
documented here as `0.12.4` (the version under test) but
cannot be enforced through Mend's provisioning mechanism.
Operators must ensure uv 0.12.4+ is available in the scan
environment.

## References

- uv 0.12.4 release notes:
  https://github.com/astral-sh/uv/releases/tag/0.12.4
- Pattern catalog entry: `uv-0124-combined` in
  `plugins/python-uv/skills/uv-core/references/feature-coverage-patterns.md`
