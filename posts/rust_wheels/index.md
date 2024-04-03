---
title: Precompiling Rust for Python Wheel Distribution
description: |
    This blog post relates to the addition of rust integration and automating build releases.
categories: [devops, python, rust, github]
date: 4/03/2024
code-line-numbers: false
draft: true
---

If the world was perfect everybody would develop exclusively on debian-based OS. A work in progress project of mine is developing a fully-featured python repository template. This blog post relates to the addition of rust integration and automating build releases.

I am going to detail the modifications I made to my template repository. <https://github.com/evmckinney9/python-template>

Assumes basic knowledge of github, github action, python packaging and some familiarity with integrating rust into python projects.

- **Official **: description and [link](url)
- 

Distributing Python packages that contain Rust extensions poses a challenge, especially when targeting machines where Rust is not installed. This typically results in a failed installation as the package cannot be built from source. To overcome this, we utilize GitHub Actions to build wheels for multiple platforms, thus enabling distribution to machines without requiring Rust. To distribute Python packages with Rust extensions to machines lacking Rust, we employ GitHub Actions with `cibuildwheel`. This approach automates the creation of wheels across multiple platforms.

___
setuptools-rust build wheels github action

official docs: <https://doc.rust-lang.org/stable/book/>

google course on rust: <https://google.github.io/comprehensive-rust/>

misc github resources on rust:

- <https://github.com/rust-lang/rustlings>
- <https://github.com/rust-unofficial/awesome-rust>
- <https://github.com/TheAlgorithms/Rust>

rust with python:
- main ref: <https://pyo3.rs>
- popular build tool (I don't use): <https://www.maturin.rs/>

example project (rust in qiskit): [https://github.com/Qiskit/qiskit/tree/main/crates/accelerate](https://github.com/Qiskit/qiskit/tree/main/crates/accelerate)

- [https://medium.com/qiskit/new-weve-started-using-rust-in-qiskit-for-better-performance-a3676433ca8c](https://medium.com/qiskit/new-weve-started-using-rust-in-qiskit-for-better-performance-a3676433ca8c)


# Motivation

Rust + python projects. I do all my development exclusively on linux (often WSL) so managing the rust compiler is pretty easy. But if I want to distribute my python module to others who only have dev setups ready to handle python than I should just precompile - distribute the wheels!

So can we make a rust+python project that we can pip install without needing rust on our machine?

# Part 1. Setup Python with Rust

Using my python boilerplate template repo as a starting place.

Pyo3 docs: <https://pyo3.rs> Wrapping rust code for us in Python.

Example: <https://github.com/PyO3/pyo3/tree/main/examples/setuptools-rust-starter>

Changes:

1. In pyproject.toml, we add `"setuptools_rust"` to build_system requirements.
Reference: <https://setuptools-rust.readthedocs.io/en/latest/>

```toml
[build-system]
requires = ["setuptools", "wheel", "setuptools-rust"]
build-backend = "setuptools.build_meta" 
```

Some additional settings

reference for RustExtension in <https://setuptools-rust.readthedocs.io/en/latest/reference.html>

```toml
[[tool.setuptools-rust.ext-modules]]
# Private Rust extension module to be nested into the Python package
# https://github.com/PyO3/setuptools-rust?tab=readme-ov-file
target = "{{project_name}}._accelerate"
path = "crates/Cargo.toml"
binding = "PyO3"
```

The last part of the target name (e.g. "rust") should match lib.name in Cargo.toml,

but you can add a prefix to nest it inside of a parent Python package or namespace.

Note that lib.name may not be defined in the Cargo.toml, but you still have to match the name of the function with the `#[pymodule]` attribute.

????????

2. MANIFEST.in, tell setuptools where the rust files are.
Ref: <https://setuptools.pypa.io/en/latest/userguide/miscellaneous.html>

```ini
include pyproject.toml
include crates/Cargo.toml
recursive-include crates *.rs
recursive-include src *
```

3. crates/Cargo.toml
Reference: <https://doc.rust-lang.org/cargo/reference/manifest.html>

```toml
[package]
name = "{{project_name}}_accelerate"
version = "0.1.0"
edition = "2021"

[lib]
name = "{{project_name}}_accelerate"
crate-type = ["cdylib"]
path = "src/lib.rs"

[dependencies.pyo3]
version = "0.20.3"
features = ["extension-module"]
```

in crates/src/basic_functions/basic_math.rs I define the functions I want

```rust
// crates/src/basic_functions/basic_math.rs
use pyo3::prelude::*;

#[pyfunction]
#[pyo3(text_signature = "(a, b, /)")]
pub fn add_in_rust(a: i32, b: i32) -> PyResult<i32> {
    Ok(a + b)
}

#[pyfunction]
#[pyo3(text_signature = "(a, b, /)")]
pub fn subtract_in_rust(a: i32, b: i32) -> PyResult<i32> {
    Ok(a - b)
}
```

and then use mod.rs to package the functions into the basic_functions module

```rust
// crates/src/basic_functions/mod.rs
pub mod basic_math;
pub mod basic_strings;

use pyo3::prelude::*;

#[pymodule]
pub fn basic_functions(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_wrapped(wrap_pyfunction!(basic_math::add_in_rust))?;
    m.add_wrapped(wrap_pyfunction!(basic_math::subtract_in_rust))?;
    m.add_wrapped(wrap_pyfunction!(basic_strings::concat_in_rust))?;
    Ok(())
}
```

and lib.rs to expose the modules to python

```rust
// crates/src/lib.rs

use pyo3::prelude::*;
use pyo3::wrap_pymodule;

mod basic_functions;

#[pymodule]
fn _accelerate(_py: Python<'_>, m: &PyModule) -> PyResult<()> {
    m.add_wrapped(wrap_pymodule!(basic_functions::basic_functions))?;
    Ok(())
}
```

which uses wrap_pymodule to…

Now when we instantiate the template repository, and build.

![Figure 1: Caption.](figures/image1.png)  

Then I clone this new repository on my WSL and build the project. Recall, from my project template the command `make init` will create a virtual environment and install the project with all dependencies.

![Figure 2: Caption.](figures/image2.png)  

Great. looks like it works, as expected since this notebook allows me to import both python module and rust module code and execute it successfully.

![Figure 3: Caption.](figures/image3.png)  

# Part 2. Distributing Wheels

info about github actions:

- <https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners>
- <https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-python>

Switch to windows (machine without Rust installed)

```bash
evmck@Evan-Desktop ~/Downloads/windows_rust_demo
$ pip install -e git+https://github.com/evmckinney9/rust_demo#egg=rust_demo
```

and fails

```text
ERROR: Failed building editable for rust_demo
Failed to build rust_demo
ERROR: Could not build wheels for rust_demo, which is required to install pyproject.toml-based projects 
(.venv) 
```

That's a shame. How could we have modified our project's release so it could have been distributed to the rust-less machine?

Common strategy is to use actions to build distribution packages (and optionally publish to PyPI). So let's piece together a release workflow using github actions.

Information about github-hosted runners: <https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners/about-github-hosted-runners>

## Building Wheels for Multiple Platforms

We need to configure Github Action with cibuildwheels: <https://cibuildwheel.pypa.io/en/stable/>

Works with Github Actions to build wheels for Linux, macOS, Windows.

<https://cibuildwheel.pypa.io/en/stable/setup/#github-actions>

```yaml
name: Build

on: [push, pull_request]

jobs:
  build_wheels:
    name: Build wheels on ${{ matrix.os }}
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        # macos-13 is an intel runner, macos-14 is apple silicon
        os: [ubuntu-latest, windows-latest, macos-13, macos-14]

    steps:
      - uses: actions/checkout@v4

      - name: Build wheels
        uses: pypa/cibuildwheel@v2.17.0

	  - uses: actions/upload-artifact@v4
        with:
          name: cibw-wheels-${{ matrix.os }}-${{ strategy.job-index }}
          path: ./wheelhouse/*.whl
```

This configuration automates the build process for Linux, macOS, and Windows, ensuring your package is accessible across these platforms.

Let's look at the pre-installed tools for each runner: <https://github.com/actions/runner-images#available-images>

and I confirmed that python and rust come pre-installed. However, even though rust is installed I found I needed to include these environment variables. See: <https://github.com/pypa/cibuildwheel/issues/1684>.

> The linux build runs in a [container](https://github.com/pypa/manylinux), not on the host machine. That container doesn't have the rust toolchain installed, so you have to install it.

```yaml
env:
  CIBW_BEFORE_BUILD_LINUX: curl -sSf https://sh.rustup.rs | sh -s -- -y
  CIBW_ENVIRONMENT_LINUX: "PATH=$HOME/.cargo/bin:$PATH"
  CIBW_SKIP: "cp36-* cp37-* cp38-* cp39-* pp* *-win32 *-musllinux* *_i686"
```

Note, we also include CIBW_SKIP, because issue (<https://github.com/rust-lang/rustup/issues/2984>) not all linux platforms are fully compatible with the rust installation.

## Building Wheels for Multiple Python Versions

"By default, Python extension modules can only be used with the same Python version they were compiled against. The advantage of building extension modules using the limited Python API is that package vendors only need to build and distribute a single copy (for each OS / architecture), and users can install it on all Python versions from the minimum version and up."

Reference:

<https://pyo3.rs/v0.21.0/building-and-distribution.html>

> [`setuptools-rust`](https://github.com/PyO3/setuptools-rust) is an add-on for `setuptools` which adds extra keyword arguments to the `setup.py` configuration file. It requires more configuration than `maturin`, however this gives additional flexibility for users adding Rust to an existing Python package that can't satisfy `maturin`'s constraints.

<https://setuptools-rust.readthedocs.io/en/latest/building_wheels.html>

> Because `setuptools-rust` is an extension to `setuptools`, the standard [`python -m build`](https://pypa-build.readthedocs.io/en/stable/) command (or [`pip wheel --no-deps . --wheel-dir dist`](https://pip.pypa.io/en/stable/cli/pip_wheel/)) can be used to build distributable wheels. These wheels can be uploaded to PyPI using standard tools such as [twine](https://github.com/pypa/twine).

There are three steps involved in making use of `abi3` when building Python packages as wheels:

> 1. Enable the `abi3` feature in `pyo3`. This ensures `pyo3` only calls Python C-API functions which are part of the stable API, and on Windows also ensures that the project links against the correct shared object (no special behavior is required on other platforms):

So I add this to my `crates/Cargo.toml`

```toml
[dependencies.pyo3]
version = "0.20.3"
features = ["extension-module", "abi3"]
```

> 2. Ensure that the built shared objects are correctly marked as `abi3`. This is accomplished by telling your build system that you're using the limited API. [`maturin`](https://github.com/PyO3/maturin) >= 0.9.0 and [`setuptools-rust`](https://github.com/PyO3/setuptools-rust) >= 0.11.4 support `abi3` wheels. See the [corresponding](https://github.com/PyO3/maturin/pull/353) [PRs](https://github.com/PyO3/setuptools-rust/pull/82) for more.
>
> 3. Ensure that the `.whl` is correctly marked as `abi3`. For projects using `setuptools`, this is accomplished by passing `--py-limited-api=cp3x` (where `x` is the minimum Python version supported by the wheel, e.g. `--py-limited-api=cp35` for Python 3.5) to `setup.py bdist_wheel`.

Do both steps by add this to `pyproject.toml` (relevant closed issue: <https://github.com/PyO3/setuptools-rust/issues/399>). Note, the official documentation suggests using either setup.cfg or `DIST_EXTRA_CONFIG` environment variable. (I'm not entirely sure this is working still because I don't see this flag as I expect in the action logs.)

```toml
[tool.distutils.bdist_wheel]
py_limited_api = "cp39"
```

## Generate Tagged Release Notes

Reference:

- conventional commits(<https://www.conventionalcommits.org>)
- angular commit convention (<https://github.com/angular/angular/blob/68a6a07/CONTRIBUTING.md#commit>)
- Popular tool is <https://github.com/vaab/gitchangelog>

Enforce conventional commits to automatically generate CHANGELOGs, and semantic versioning. but because I have already set up enforcement of using this commit-msg hook <https://github.com/qoomon/git-conventional-commits> there is a better way.

I use this tool <https://github.com/conventional-changelog/conventional-changelog> to build the changelog notes. This tool is intended to work by generating the release notes before the tagged commitbut because I use the tagged commit to trigger this action we need to do something a bit messy.

Specify `-r 2` to get the last two releases (the most recent commit is from HEAD ahead of the last tagged commit, but we really want the second one). Then trim the first two lines of this file to remove the junk notes.

```yaml
- name: Generate Changelog
run: |
  npm install -g conventional-changelog-cli
  conventional-changelog -p angular -r 1 > temp_changelog.md
  echo "$(cat temp_changelog.md CHANGELOG.md)" > CHANGELOG.md

- name: Commit changes and push back to GitHub
uses: stefanzweifel/git-auto-commit-action@v4
with:
  commit_message: "docs(changelog): update CHANGELOG.md for release"
  file_pattern: CHANGELOG.md
  branch: main
```

I make this separation because on the release, I can link the path for the release notes to be the output for the last tag while still maintaining a separate file for all changes.

GH-release action (<https://github.com/marketplace/actions/gh-release>)

```yaml
- name: Create Release
uses: softprops/action-gh-release@v2
with:
  files: ./wheelhouse/*.whl
  body_path: temp_changelog.md
env:
  GITHUB_TOKEN: '${{ secrets.GITHUB_TOKEN }}'
```

I'll also highlight Google's release-please action, <https://github.com/googleapis/release-please,> which generates release PRs based on conventional commits. This could be a good option; however, python repositories must have a `setup.py` and `setup.cfg` to work.

This is also a good option, but this will only generate release notes based on merged pull requests instead of by commits. This is in some ways better - but my repositories do not necessarily enforce branch protection rules, which means you can directly commit to `main`.

<https://docs.github.com/en/repositories/releasing-projects-on-github/automatically-generated-release-notes#about-automatically-generated-release-notes>

## Putting it All together

Here are two example projects that use cibuildwheel with setuptools_rust

1. <https://github.com/daggy1234/polaroid/blob/main/.github/workflows/publish.yml>
2. <https://github.com/etesync/etebase-py/blob/master/.github/workflows/manual.yml>

Here is the file from the template repository:
https://github.com/evmckinney9/python-template/blob/main/.github/workflows/release.yml

```yaml
name: CI

on: # triggered by push tagged commits to main
  push:
    tags:
      - 'v*'

permissions:
  contents: write

jobs:
  check-for-flag-file:
    if: github.repository != 'evmckinney9/python-template'
    runs-on: ubuntu-latest
    outputs:
      continue: ${{ steps.flag-check.outputs.continue }}
    steps:
      - uses: actions/checkout@v4
      - name: Check for template_flag.yml
        id: flag-check
        run: |
          if [ ! -f .github/template_flag.yml ]; then
            echo "continue=true" >> $GITHUB_OUTPUT
          else
            echo "continue=false" >> $GITHUB_OUTPUT
          fi

  build_wheels:
    needs: check-for-flag-file
    if: needs.check-for-flag-file.outputs.continue == 'true'
    name: 'Build wheels on ${{ matrix.os }}'
    runs-on: '${{ matrix.os }}'
    strategy:
      matrix:
        os:
          - ubuntu-latest
          - windows-latest
          - macos-13
          - macos-14
    env:
      CIBW_BEFORE_BUILD_LINUX: curl -sSf https://sh.rustup.rs | sh -s -- -y
      CIBW_ENVIRONMENT_LINUX: "PATH=$HOME/.cargo/bin:$PATH"
      CIBW_SKIP: "cp36-* cp37-* cp38-* cp39-* pp* *-win32 *-musllinux* *_i686"

    steps:

      - uses: actions/checkout@v4
      - name: Check if template_flag.yml is gone
        id: check-template
        if: ${{ hashFiles('.github/template_flag.yml') == '' }}
        run: |
          echo "release_continue=true" >> $GITHUB_ENV

      - name: Build wheels
        uses: pypa/cibuildwheel@v2.17.0
      - uses: actions/upload-artifact@v4
        with:
          name: 'cibw-wheels-${{ matrix.os }}-${{ strategy.job-index }}'
          path: ./wheelhouse/*.whl
  release:
    needs: [check-for-flag-file, build_wheels]
    if: needs.check-for-flag-file.outputs.continue == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Check out the repo
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/checkout@v4
      - name: Check for initial commit condition and exit if true
        run: |
          if [ -f .github/template_flag.yml ]; then
            echo "Initial commit setup detected, exitting."
            exit 0
          fi
      - name: Set up Node.js
        uses: actions/setup-node@v4

      - name: Generate Changelog
        run: |
          npm install -g conventional-changelog-cli
          conventional-changelog -p conventionalcommits -r 2 | tail -n +3 > CHANGELOG.tmp

      - uses: actions/download-artifact@v4
        with:
          path: ./wheelhouse

      - name: Create Release
        uses: softprops/action-gh-release@v2
        with:
          files: ./wheelhouse/**/*.whl
          body_path: CHANGELOG.tmp
        env:
          GITHUB_TOKEN: '${{ secrets.GITHUB_TOKEN }}'
```

## Verification

Let's make sure using the template now works. Can we commit a change, create a release, and pip install on windows (with no rust) and on a lower python version.

![Figure 4: Caption.](figures/image4.png)  

and after some tweaking got this working

![Figure 5: Caption.](figures/image5.png)  

Let's try installing again. Instead of

```bash
pip install -e git+https://github.com/evmckinney9/template_demo#egg=template_demo
```

I'll directly install the wheel file from the latest release.

```bash
pip install  https://github.com/evmckinney9/template_demo/releases/download/v0.2.0/template_demo-0.1.0-cp39-abi3-win_amd64.whl
```

Success!! The `template_demo` package was successfully installed - and I could run the rust methods from `_accelerate` even though I never installed Rust on my windows machine.

## Next

- Dockerfile or Containerfile -> make it easier to reproduce experiments
- Remove hardcoding of python-version across multiple files. Use a .py-version file?
- Improve version bumping. Enforce semantic versioning and something that updates the pyproject.toml file
- generate documentation (sphinx, mkdocs, pdocs) Want something simple and integrates best with my existing documentation style (google docstrings)
- Find a setup for debugging template actions. Current approach is update template -> clone template -> commit to trigger action -> observe what happens. Can I trigger actions in a more controlled manner before having to push changes back to github?
