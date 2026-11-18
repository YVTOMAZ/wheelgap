# FastAPI: CPython 3.15 test dependencies across three platforms

[FastAPI](https://github.com/fastapi/fastapi) is an asynchronous Python web framework. This case study checks its locked **test environment**, not just its production dependencies, at revision [`b2831196bfee3eb9ce303a3c6440f2a104832954`](https://github.com/fastapi/fastapi/tree/b2831196bfee3eb9ce303a3c6440f2a104832954). The scan used **wheelgap 0.13.0** and package files available on **18 November 2026**.

## Targets and selection

FastAPI's pinned [CI test matrix](https://github.com/fastapi/fastapi/blob/b2831196bfee3eb9ce303a3c6440f2a104832954/.github/workflows/test.yml) includes CPython 3.15 and free-threaded 3.15t on Ubuntu, Windows and macOS runners. This study checks the regular 3.15 jobs (see [Free-threaded 3.15t](#free-threaded-315t) for the others). We check:

- `linux-x86_64` and `win-amd64`, corresponding to the x64 Linux and Windows runner families.
- `macos-arm64` as an illustrative Apple-silicon target; the workflow's `macos-latest` label alone does not establish that runner's architecture.

The CI installation uses `uv sync --no-dev --group tests --extra all`. The wheelgap command uses the same dependency selection against the [pinned `uv.lock`](https://github.com/fastapi/fastapi/blob/b2831196bfee3eb9ce303a3c6440f2a104832954/uv.lock). This includes test dependencies and extras and should not be read as an audit of FastAPI's minimal runtime footprint.

## Reproduce the scan

From the pinned FastAPI checkout:

```bash
python -m pip install wheelgap==0.13.0
wheelgap uv.lock -p 3.15 \
  -t linux-x86_64,win-amd64,macos-arm64 \
  --no-dev --group tests --extra all --explain --suggest
```

Trimmed output (the many rejected wheel filenames and the `macos-arm64` `--explain` block, which reads like the other two, are omitted):

```text
wheelgap 0.13.0 · uv.lock · 115 packages · Python 3.15

package  version  linux-x86_64  win-amd64  macos-arm64
pyyaml   6.0.3    ● source      ● source   ● source

114 packages with wheels for every target hidden (--all to show)

pyyaml 6.0.3 on linux-x86_64: ● source
  via: fastapi → pyyaml
  pyyaml-6.0.3-cp314-cp314-manylinux2014_x86_64.manylinux_2_17_x86_64.manylinux_2_28_x86_64.whl
  built for cp314-cp314, not CPython 3.15

pyyaml 6.0.3 on win-amd64: ● source
  via: fastapi → pyyaml
  pyyaml-6.0.3-cp314-cp314-win_amd64.whl
  built for cp314-cp314, not CPython 3.15

pyyaml 6.0.3: no newer release has a wheel for linux-x86_64, win-amd64, macos-arm64

skipped 1 non-index package: fastapi (editable)
linux-x86_64  1 from source
win-amd64     1 from source
macos-arm64   1 from source
1 of 115 packages has gaps: 1 will build from source.
```

The scan exits **1**, which wheelgap uses to report a wheel gap, not a command failure.

## PyYAML 6.0.3: a Python-version gap

The locked release has CPython 3.14 and earlier wheels for these operating systems, but not CPython 3.15 wheels. A platform-matching wheel tagged `cp314-cp314` cannot be used by `cp315`. A source distribution is available, so wheelgap says `source`, not `missing`. The `fastapi` editable workspace package is skipped separately.

With `--suggest`, wheelgap found **no newer PyYAML release** with a compatible wheel for all three checked targets at the time of the scan. Reasonable choices are to validate building PyYAML from source in the relevant CI jobs, use a supported Python version with existing wheels when appropriate, or move to a future PyYAML release with compatible wheels once available and tested. This observation does not imply that FastAPI itself fails to install or that PyYAML cannot compile.

## Free-threaded 3.15t

The same matrix also runs 3.15t, which this scan doesn't cover. The pinned `uv.lock` suggests that target has more gaps: `argon2-cffi-bindings`, `psutil`, `watchfiles` and `ast-serialize` list only `cp314t` and `abi3` wheels for these platforms, and a free-threaded build can't use `abi3` wheels. PyYAML has no `cp315t` wheel either. To check it, run the command above with `-p 3.15t`.

## Independent pip checks

The study also used pip **26.3** against public PyPI, with `--no-deps --only-binary=:all: --python-version 3.15 --implementation cp --abi cp315`. For `PyYAML==6.0.3` and `PyYAML>=6.0.3`, pip returned **No matching distribution found** for `win_amd64`, `macosx_11_0_arm64` and `manylinux_2_28_x86_64`. The Linux check was for that specific manylinux tag, rather than pip's complete set of compatible older glibc tags; wheelgap checked the full target compatibility.

For example, check the Windows wheel availability without installing the package:

```bash
python -m pip --isolated download --no-deps --only-binary=:all: \
  --python-version 3.15 --implementation cp --abi cp315 \
  --platform win_amd64 --dest wheelhouse "PyYAML==6.0.3"
```

Both checks inspect published files. They do not execute any dependency's build or test FastAPI's application behaviour.
