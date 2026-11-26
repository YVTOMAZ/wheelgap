# Rust extensions (PyO3 + maturin)

When a package grows a Rust extension, it stops being a `py3-none-any`
wheel: the compiled extension is tied to a Python ABI and a platform, so pip
can only install a prebuilt wheel that matches the interpreter and the
machine. Without one, pip falls back to the sdist and compiles the Rust on
the user's machine, which needs a Rust toolchain and a matching compiler.
This page is the checklist for shipping those wheels.

## Why it matters

A pure-Python package publishes one `py3-none-any` wheel and every platform
installs it. Add a Rust extension and that single wheel no longer works:
the compiled `.so`/`.pyd` is built for one Python ABI and one OS/arch/libc,
so each platform you support needs its own wheel. Miss one and the users on
that platform either compile from source (Rust toolchain required, slow,
fragile) or get no install at all.

## The matrix to build

Build one wheel per row. `windows-11-arm` is GitHub's native ARM64 Windows
runner; the Linux rows need both a glibc (manylinux) and a musl
(musllinux) build.

| Platform | Runner | wheelgap target |
| --- | --- | --- |
| Linux x86_64 (manylinux) | `ubuntu-latest` | `linux-x86_64` |
| Linux x86_64 (musllinux) | `ubuntu-latest` (musl container) | `musl-x86_64` |
| Linux aarch64 (manylinux) | `ubuntu-24.04-arm`, or QEMU | `linux-aarch64` |
| Linux aarch64 (musllinux) | `ubuntu-24.04-arm` (musl container) | `musl-aarch64` |
| macOS x86_64 | `macos-13` | `macos-x86_64` |
| macOS arm64 | `macos-14` or newer | `macos-arm64` |
| Windows x64 | `windows-latest` | `win-amd64` |
| Windows arm64 | `windows-11-arm` | `win-arm64` |

Use [PyO3/maturin-action](https://github.com/PyO3/maturin-action) to drive
maturin on each runner. A minimal build step:

```yaml
- uses: PyO3/maturin-action@v1
  with:
    command: build
    args: --release -i python3.10 -i python3.11 -i python3.12 -i python3.13 -i python3.14 -i python3.14t
```

maturin and the Action handle the manylinux container and cross-compilation;
see the Action's README for `target`, `manylinux`, `container` and
`rust-toolchain`, and the [maturin book](https://www.maturin.rs/) for local
builds and `maturin generate-ci`. Linking instead of repeating them keeps
this page short.

## Pythons

Pass the interpreters explicitly with `-i` / `--interpreter` instead of
`--find-interpreter`, so the set you ship is the set you check:

```bash
maturin build --release \
  -i python3.10 -i python3.11 -i python3.12 -i python3.13 -i python3.14 -i python3.14t
```

!!! note
    An `abi3` wheel does not install on a free-threaded interpreter:
    free-threaded CPython needs `cp314-cp314t` wheels (or `abi3t` on 3.15+).
    If you ship `abi3` only, pass `3.14t` explicitly so the gap shows up before
    release, not in a user's install. See
    [Wheels on free-threaded Python](free-threading.md).

## Check before you publish

Run wheelgap on the `dist/` folder before uploading, with the Pythons and
targets you actually ship:

```bash
uvx wheelgap --dist dist/ -p 3.10 -p 3.11 -p 3.12 -p 3.13 -p 3.14 -p 3.14t \
  -t linux-x86_64,linux-aarch64,musl-x86_64,musl-aarch64,macos-x86_64,macos-arm64,win-amd64,win-arm64
```

It checks each file in `dist/` against the targets, compares with the last
release on PyPI, and fails if a target you had a wheel for got none this
time. In GitHub Actions the same check is the `dist` input:

```yaml
- uses: wheelgap/wheelgap@v0.14.0
  with:
    dist: dist
    args: -p 3.10 -p 3.11 -p 3.12 -p 3.13 -p 3.14 -p 3.14t -t linux-x86_64,linux-aarch64,musl-x86_64,musl-aarch64,macos-x86_64,macos-arm64,win-amd64,win-arm64
```

See [Before a release](ci.md#before-a-release) and
[`--dist`](cli.md#dist) for what a dropped target means and the exit code.
