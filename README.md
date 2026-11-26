<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2025 The Linux Foundation
-->

# 🐍 Python Project Dynamic Versioning

> [!WARNING]
> **Deprecated:** this action will get no further development. The
> repository will become archived; workflows that reference its
> existing tags and commits keep working after that.
> [`lfreleng-actions/build-metadata-action`](https://github.com/lfreleng-actions/build-metadata-action)
> replaces it: see
> [Migrating to build-metadata-action](#migrating-to-build-metadata-action).

Detects whether a Python project uses dynamic versioning. The action
recognises signals across the full range of legacy and modern project
layouts:

- `pyproject.toml`:
  - `[project] dynamic = ["version"]`
  - `[build-system].requires` lists `setuptools_scm` / `setuptools-scm`,
    `hatch-vcs`, or `pbr`
- `setup.cfg`:
  - A `[pbr]` section
  - `[metadata] version = attr:` or `[metadata] version = file:`
    indirection
  - `[options] setup_requires` containing `pbr`, `setuptools_scm`,
    `setuptools-scm`, or `versioneer`
- `setup.py`:
  - `pbr=True` or `setup_requires=[..., 'pbr', ...]`
  - `use_scm_version=...` or `setup_requires` referencing
    `setuptools_scm` / `setuptools-scm`
  - `versioneer.get_version` / `versioneer.get_cmdclass`

## Migrating to build-metadata-action

Use release v0.13.5 or later of
[`build-metadata-action`](https://github.com/lfreleng-actions/build-metadata-action).
Earlier releases leave `python_dynamic_provider` empty for a project
whose `pyproject.toml` declares the version dynamic
([lfreleng-actions/build-metadata-action#226](https://github.com/lfreleng-actions/build-metadata-action/issues/226)).

Replace the steps that called this action with:

<!-- markdownlint-disable MD013 -->

```yaml
- name: 'Check for dynamic project versioning'
  id: dyn
  uses: lfreleng-actions/build-metadata-action@0000000000000000000000000000000000000000  # REPLACE: vX.Y.Z
  with:
    project_type: 'python'
    use_version_extract: 'false'
    include_environment: 'false'
    artifact_upload: 'false'
    output_format: ''
    python_offline_mode: 'true'

- name: 'Branch on the result'
  env:
    VERSIONING_TYPE: ${{ steps.dyn.outputs.python_versioning_type }}
    DYNAMIC_PROVIDER: ${{ steps.dyn.outputs.python_dynamic_provider }}
  run: |
    echo "versioning type:  $VERSIONING_TYPE"
    echo "dynamic provider: $DYNAMIC_PROVIDER"
```

<!-- markdownlint-enable MD013 -->

- Pin the action to the commit SHA of the release you choose, in place
  of the placeholder.
- `project_type: 'python'` stops a `package.json` or `go.mod` beside
  the Python files from naming a different project.
- The other five inputs keep the step as quiet as this action: no
  version lookup, no probe of the runner, no uploaded artefact, no job
  summary and no query for current Python releases.
- Add `path_prefix` when the project does not sit at the repository
  root.
- The second step passes the outputs through `env:` rather than
  expanding them inside `run:`, which keeps their values out of the
  shell script.

Expect the step to take longer than this action did: it sets up Go and
builds the action's binary on each run.

### Output mapping

<!-- markdownlint-disable MD013 -->

| This action                          | `build-metadata-action`                         |
| ------------------------------------ | ----------------------------------------------- |
| `dynamic_version`: `true` or `false` | `python_versioning_type`: `dynamic` or `static` |
| `dynamic_provider`                   | `python_dynamic_provider`                       |
| `source`                             | none                                            |

<!-- markdownlint-enable MD013 -->

`python_dynamic_provider` takes the same six values as
`dynamic_provider` (`pbr`, `setuptools-scm`, `versioneer`,
`setuptools-dynamic`, `hatch-vcs` and `pyproject-dynamic`), from the
same signals, and stays empty for a static version.

### Differences

1. **A seventh provider.** `build-metadata-action` also reports
   `runtime-attr`, with `python_versioning_type` set to `dynamic`, for a
   `setup.py` that takes its version from `__version__`, `get_version`
   or `read_version`. This action reports such a project as static.
2. **Where the search stops.** When `pyproject.toml` gives no signal,
   this action goes on to `setup.cfg` and then `setup.py`.
   `build-metadata-action` reads no further once `pyproject.toml` names
   the project. A project with a static `[project]` version and a
   leftover `[pbr]` section in `setup.cfg` gets `true` and `pbr` here,
   but `static` from `build-metadata-action`.
3. **No `source` output.** `python_metadata_source` comes nearest, but
   it names the file that supplied the project name, not the
   versioning signal. For a PBR project that keeps its name in
   `setup.cfg` and `pbr=True` in `setup.py`, this action reports
   `setup.py` while `python_metadata_source` gives `setup.cfg`.
4. **No project metadata.** For a `pyproject.toml` holding a
   `[build-system]` table alone, with no `setup.cfg` or `setup.py`
   beside it, this action reports `false`, while
   `build-metadata-action` leaves `python_versioning_type` empty.

## Inputs

<!-- markdownlint-disable MD013 -->

| Input         | Required | Default | Description                           |
| ------------- | -------- | ------- | ------------------------------------- |
| `path_prefix` | False    | `.`     | Path/directory to Python project code |

<!-- markdownlint-enable MD013 -->

## Outputs

<!-- markdownlint-disable MD013 -->

| Output             | Description                                                                                                                                             |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `dynamic_version`  | `true` when the project uses dynamic versioning, `false` otherwise.                                                                                     |
| `dynamic_provider` | Identifier of the provider: `pbr` \| `setuptools-scm` \| `versioneer` \| `setuptools-dynamic` \| `hatch-vcs` \| `pyproject-dynamic`. Empty when static. |
| `source`           | Path to the file that supplied the dynamic-versioning signal. Empty when the project declares no dynamic-versioning signal.                             |

<!-- markdownlint-enable MD013 -->

## Implementation

The action delegates detection to `scripts/detect_dynamic_version.py`,
a small Python helper bundled with the action. The helper uses the
standard library's `tomllib` for `pyproject.toml` and `configparser`
for `setup.cfg`, and falls back to regex for `setup.py`. The full
unit-test suite lives under `tests/`.

The action provisions its own Python 3.11 interpreter with
`actions/setup-python` (`update-environment: false`, so the caller's
`PATH` stays unchanged) and runs the helper in isolated mode (`-I`),
rather than using whichever `python3` the caller's `PATH` resolves.
