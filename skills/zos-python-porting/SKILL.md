---
name: zos-python-porting
description: Load before porting Python packages to z/OS with zopen. Covers pyproject/setup metadata, check_python dependency mapping, C-extension detection, --c-extensions generation, wheel build/check behavior across the 3.12/3.13/3.14 interpreter matrix, the zopen_python_test hook, pytest import shadowing, native test runners, test-result parsing that sums across interpreters, prebuilt Rust wheels via BARE ports, z/OS shell and encoding traps, wheel publishing, repository creation, and CI pipeline creation.
---

# z/OS Python Porting

Use this skill for end-to-end Python package porting work with local `zopen-*` commands.

## Core Rules

1. Use local CLI commands directly (`zopen-generate`, `zopen-build`, `zopen-info`, etc.).
2. Use `--help` as source of truth for flags/syntax in installed tooling.
3. Prefer Homebrew formula metadata and upstream project metadata first; use web search only as fallback.
4. Do not create files in `patches/` until build succeeds.
5. **Always run `zopen-build` in foreground with appropriate timeout, never as background process.** This ensures proper error capture and debugging.
6. After the port builds and validation passes, create the zopencommunity repository and CI/CD job by default. Do not treat repo/CI setup as optional unless the user explicitly says to skip it.
7. Before finalizing, verify tests are actually processed by `zopen_check_results` and the parsed test success rate is greater than 90%. If no tests are parsed or the success rate is 90% or lower, fix the test flow or get explicit user approval before continuing.

## CRITICAL: Continuous Skill Improvement

**If the user provides a course correction, tip, or alternative approach that successfully resolves an issue, you MUST update this `SKILL.md` file in the `https://github.com/zopencommunity/ai-porting` repository.** This ensures the skill evolves and prevents future agents from repeating the same mistakes. Treat this as a core mandate of the porting workflow.

## Preflight

Run:

```bash
command -v zopen-generate zopen-build zopen-info zopen-query zopen-version zopen-create-repo zopen-create-cicd-job jq git bump
zopen-generate --help
zopen-build --help
zopen-info --help
zopen-create-repo --help
zopen-create-cicd-job --help
zopen-version || zopen-version --help
```

If a command reports `Source the zopen-config prior to running ...`, source zopen config first.

## MCP Integration

**For porting issues, leverage the `zos-porting-rag` MCP server** which provides access to 600+ z/OS porting patches and documentation. 

**Setup**: See [zos-porting-rag repository](https://github.com/ZOSOpenTools/zos-porting-rag) for MCP server configuration.

## Workflow

### 1. Collect Metadata

Gather:
- project name (lowercase)
- one-line description
- repo URL (prefer the Git clone repository location over an archive file)
- SPDX license
- categories
- build system
- stable URL + dependencies

Use:
- `zopen-generate --list-licenses`
- `zopen-generate --list-categories`
- `zopen-generate --list-build-systems`

## Python Porting

### Rust Extensions Without a Native z/OS Toolchain

Some Python packages are Rust extensions (pydantic-core, rpds-py, watchfiles).
There is **no native Rust toolchain for z/OS**, so they cannot be compiled by
`ZOPEN_BUILD_SYSTEM=Python` on-platform.

**Before building cross-compile infrastructure, check whether you need the
package at all.** Its only consumer may accept a pure-Python alternative — e.g.
`fastapi<=0.125.0` accepts `pydantic>=1.7.4,<3.0.0`, and pydantic v1 is pure
Python, so pinning it removes the Rust dependency entirely (`fastapi>=0.130.0`
requires pydantic v2 and therefore pydantic-core). Resolve the full closure
first and confirm how much is genuinely impure: for both fastapi and fastmcp,
**pydantic-core is the only non-pure package** out of ~13.

**You cannot repackage one `.so` across interpreters.** pydantic-core's ABI tag
is `cp312`, not `abi3`: it is compiled against CPython 3.12's private ABI, whose
struct layouts change between minor versions. Renaming it to `cpython-313.so`
yields something 3.13 loads and then corrupts memory in. Upstream ships 14
wheels *per interpreter* for this reason, and `Cargo.toml` does not enable
PyO3's `abi3` feature. Cross-compile once per interpreter, or ship one and pin
`ZOPEN_PYTHON_VERSIONS`.

**Solution**: cross-compile on Linux-on-Power using IBM's Rust cross-compiler
targeting `s390x-ibm-zos` (`cross/build_zos_wheel.py` in
`github.ibm.com/compiler/rust-scripts`), attach the wheels as release assets on
the port repo, and let the port download and verify them.

Use `ZOPEN_TYPE="BARE"`, as `fdport` does. Requirements that are easy to miss:

- **`ZOPEN_STABLE_URL` is still required.** `checkEnv` exempts only
  `ZOPEN_TYPE=LOCAL`; a BARE port without one fails with `Building from stable,
  but ZOPEN_STABLE_URL not specified` before doing anything. Nothing is fetched
  from it — `getCode` returns early for BARE — so it records which upstream
  source the binary was built from.
- **Supply `zopen_append_to_env` yourself.** The `PYTHONPATH` snippet is only
  auto-generated for `ZOPEN_BUILD_SYSTEM="Python"` ports. Without it the module
  installs and is not importable, and install validation fails.
- **Stage to `$ZOPEN_ROOT/install/dist/`**, never `$ZOPEN_INSTALL_DIR/dist/`.
- **`chtag -b` the wheel.** It is a zip; tagging it as text invites conversion.
- **Derive the interpreter tag from Python, not by parsing `--version`:**
  `python3 -c 'import sys; print("%d%d" % sys.version_info[:2])'`. z/OS grep has
  no `-o`, and IBM's banner puts the version in a different field from upstream
  CPython's.
- **Upload wheels under their native name**
  (`pkg-1.0-cp312-cp312-os390_29_00_8561.whl`) and let the port retag them.
  Renaming to `-none-any` by hand leaves the internal `WHEEL` metadata still
  declaring the original platform tag, which is malformed per PEP 427 — and it
  will not self-heal, because retagging treats a `-none-any` name with a `cp*`
  tag as already done.
- **Strip the binary** before packaging. An unstripped pydantic-core `.so` is
  ~43 MB against ~2 MB on Linux, in every wheel, pax and install.

If keeping `ZOPEN_BUILD_SYSTEM="Python"` so the wheels reach the wheel index and
get imported per interpreter, the stand-in for the build must also **create a
venv per interpreter** — the check phase runs in them, and a `ZOPEN_MAKE` that
only fills `dist/` fails with `No virtual environment at .venv-3.13`:

```sh
zopen_custom_build() {
  _zopen_python_verify_interpreters || return 1
  rm -rf dist; mkdir -p dist || return 1
  for _pv in $(_zopen_python_versions); do
    _zopen_python_ensure_venv_for "${_pv}" || return 1
    cp "wheels/pkg-${VER}-cp$(echo "${_pv}" | tr -d '.')-"*.whl dist/ || { deactivate; return 1; }
    deactivate
  done
  _zopen_python_ensure_venv_for "$(_zopen_python_primary_version)" || return 1
  _zopen_python_retag_wheel
  _rc=$?; deactivate; return ${_rc}
}
```

Then `zopen_python_test` imports the extension on each interpreter — the only
thing that shows an off-platform build actually works on this one.

**Consumers** need the zopen index and must be stopped from falling back to the
sdist:

```sh
pip install --extra-index-url https://repo.zopen.community/pypi/wheels/simple/ \
            --only-binary <rust-pkg>  <package>
```

`--only-binary` is load-bearing: with both indexes visible pip takes the highest
version, so a newer release on PyPI is fetched as an sdist and fails to compile.
Note `zopen-build` configures **no** pip index by default.

### Decision Checklist

Before generating the port, determine:
- package source: `pyproject.toml`, `setup.py`, `setup.cfg`, PyPI metadata, or upstream docs
- build backend: setuptools, hatchling, poetry-core, meson-python, maturin, or other
- extension type: pure Python, C extension, Rust extension, or mixed native dependencies
- source URL type: prefer HTTPS git clone URL; use archive URL only when git is unsuitable
- submodules: if present, use git URL and set `ZOPEN_CLONE_SUBMODULES="yes"`
- tests: pytest-compatible, native test runner, generated tests, or no usable upstream tests
- import behavior: whether tests run from source tree would shadow the installed wheel
- test gate: newest check log must produce parsed totals and a greater than 90% success rate
- publish target: pax only, wheel only, or both pax and wheel

### Metadata Sources (in order)

1. `pyproject.toml`
2. `setup.py` or `setup.cfg`
3. PyPI metadata
4. upstream docs / README
5. Homebrew formula metadata if available - `https://formulae.brew.sh/api/formula/${PROJECT}.json`

Rules:
- Prefer upstream Python packaging metadata over Brew.
- Use `check_python` for Python dependency mapping, not `python`.
- Distinguish build requirements from runtime requirements.
- If the package uses `pyproject.toml`, inspect the build backend first
  (e.g. setuptools, hatchling, poetry-core, meson-python, maturin).
- `python -m build --wheel` handles all project types (pyproject.toml, setup.py, setup.cfg) — no need for separate code paths.

### C Extensions Detection

Check for `.c` files in the project source tree. If present:
- Add `--c-extensions` flag to `zopen-generate` — this omits `ZOPEN_COMP="skip"` so the C compiler is available during the build.
- The compiler variables (CC, CFLAGS, LDFLAGS) are exported as environment variables by `zopen-build` and Python's build system (setuptools, etc.) picks them up automatically.
- `ZOPEN_MAKE_MINIMAL="yes"` is set for all Python ports — this keeps compiler flags in the environment rather than passing them as make arguments (which would break Python builds).
- **C extensions compile successfully on z/OS with `-fvisibility=default` flag in `ZOPEN_EXTRA_CFLAGS`.** Modern Python packages often don't need additional symbol visibility patches - the compiler flag is usually sufficient for all extensions to work correctly.
- **CRITICAL for C extension linking**: Must append `LIBS` to `LDFLAGS` in `zopen_init()` function for proper linking. Example: `export LDFLAGS="$LDFLAGS $LIBS"`. This ensures zoslib and other dependency libraries are linked correctly during Python wheel build.

If no C extensions:
- `ZOPEN_COMP="skip"` is set (default for Python ports without `--c-extensions`).

### Git Submodules

**For Python ports with git submodules** (e.g., bundled C libraries): Set `ZOPEN_CLONE_SUBMODULES="yes"` in `buildenv` to ensure submodules are cloned during build. Example: python-xxhash bundles xxHash C library as submodule in `deps/xxhash/`.

### Generate Command

Pure Python:
```bash
zopen-generate \
  --name <name> \
  --description "<description>" \
  --categories "<cat1 cat2>" \
  --license <spdx_or_unknown> \
  --type BUILD \
  --build-system Python \
  --stable-url "<Git clone url or Archive url>" \
  --stable-deps "check_python <other_deps>" \
  --dev-url "<Git clone url or Archive url>" \
  --dev-deps "check_python <other_deps>" \
  --build-line stable \
  --runtime-deps "check_python" \
  --non-interactive
```

Python with C extensions:
```bash
zopen-generate \
  --name <name> \
  --description "<description>" \
  --categories "<cat1 cat2>" \
  --license <spdx_or_unknown> \
  --type BUILD \
  --build-system Python \
  --c-extensions \
  --stable-url "<Git clone url or Archive url>" \
  --stable-deps "check_python <other_deps>" \
  --dev-url "<Git clone url or Archive url>" \
  --dev-deps "check_python <other_deps>" \
  --build-line stable \
  --runtime-deps "check_python" \
  --non-interactive
```

### Build Workflow

The generated `buildenv` sets `ZOPEN_BUILD_SYSTEM="Python"`. All build logic is handled **natively by `zopen-build`** — no custom functions needed in the buildenv.

`zopen-build` automatically:
- Builds, tests and publishes **one wheel per Python interpreter** (see below)
- Creates a venv per interpreter, installs `setuptools build installer wheel`
- Runs `python -m build --wheel`
- Runs `pytest -v` for testing (installs that interpreter's wheel with `--force-reinstall` first)
- Retags compiled wheels to `cp3XY-none-any`
- Installs the primary interpreter's wheel into `$ZOPEN_INSTALL_DIR/lib/python` and symlinks scripts into `bin/`
- Copies every wheel to **`$ZOPEN_ROOT/install/dist/`** for publishing
- Sets up `PYTHONPATH` in `.env`
- Merges `LIBS` into `LDFLAGS` for C extension ports (Python's build system ignores `LIBS`)
- Parses pytest output for `zopen_check_results`
- Adds `check_python` and `grep` as implicit dependencies

**Wheels go to `$ZOPEN_ROOT/install/dist/`, not `$ZOPEN_INSTALL_DIR/dist/`.**
`ZOPEN_INSTALL_DIR` is the tree that gets packaged: a wheel there ships a second
copy of everything already under `lib/python`, and CI never finds it, because
the staging step globs `install/dist/*.whl` relative to the port directory. A
port that stages to the wrong one builds green and silently publishes nothing.

### The interpreter matrix

`ZOPEN_PYTHON_VERSIONS` defaults to `"3.12 3.13 3.14"`, narrowed at startup to
whichever are actually installed. A version named **explicitly** must be present
or the build fails before compiling; a **defaulted** one that is missing is
skipped with a log line. Interpreters are located through
`ZOPEN_PYTHON_<major>_<minor>` (exported by `check_python`), then
`python<version>` on PATH, then `python3`/`python` if either reports that
version.

Consequences for every Python port:

- A compiled extension produces one wheel per interpreter, e.g.
  `pkg-1.0-cp312-none-any.whl` and `pkg-1.0-cp313-none-any.whl`.
- A pure Python wheel is built once and shared, but still tested on each.
- **Never glob `dist/*.whl`** — that hands pip every version's wheel at once and
  it refuses with `is not a supported wheel on this platform`.
- **Never hardcode `.venv`** — that is only the primary interpreter's
  environment, so the others go untested.
- `zopen_check_results` sees every interpreter's output in **one** log and must
  sum across runs (see below).

To opt out — for instance when only one prebuilt wheel exists — set it
explicitly:

```sh
export ZOPEN_PYTHON_VERSIONS="3.12"
```

### Overriding Defaults

**To change how tests are run, define `zopen_python_test`. Do not override
`ZOPEN_CHECK`.**

`zopen_python_test` is called once per interpreter, with that interpreter's
virtual environment already active and the wheel built for it already
installed. So `python` and `pip` are the right ones, and `dist/` never has to be
inspected. Supply only the command:

```sh
zopen_python_test() {
  pip install pycryptodome-test-vectors==1.0.22 || return $?
  python -m Crypto.SelfTest
}
```

Overriding `ZOPEN_CHECK` replaces the **whole loop over interpreters**, not just
the test command. Every port that has done so re-implemented it wrongly in the
same two ways — activating `.venv` (pinning to the primary) and installing
`dist/*.whl` (handing pip every version's wheel). Reach for it only when you
genuinely need to replace the loop itself.

If tests must run outside the source tree — common for C and Rust extensions,
where a source directory of the same name shadows the installed package — do
that inside the hook:

```sh
zopen_python_test() {
  test_dir="/tmp/${ZOPEN_PROJECT_NAME}_chk_$$"
  rm -rf "${test_dir}"; mkdir -p "${test_dir}" || return $?
  cp -R tests "${test_dir}/" || return $?
  (cd "${test_dir}" && python -m pytest tests/ -v)   # subshell: never leave the build elsewhere
  rc=$?
  rm -rf "${test_dir}"
  return ${rc}
}
```

Copy `tests` as a **subdirectory**, not its contents, if fixtures use paths like
`Path("tests/files/x")`.

**Never set `ZOPEN_MAKE` to a check function.** A real port did
`export ZOPEN_MAKE="zopen_custom_check"` and its build step ran the test
function — sourcing a `.venv` that does not exist yet and installing from a
`dist/` that has not been built. If the build genuinely needs replacing, write a
separate function.

**Dependencies are not bundled.** `zopen-build` installs the port's own wheel
with `--no-deps`, and the generated `.env` adds only that port's `lib/python` to
`PYTHONPATH`. A package with runtime dependencies ships a pax that cannot
import. Install the closure explicitly:

```sh
zopen_post_install() {
  pip install --disable-pip-version-check --no-warn-script-location --no-deps \
    --target "${ZOPEN_INSTALL_DIR}/lib/python" \
    <dep> <dep> ... || return $?
}
```

Dependencies needed only by the tests belong in `zopen_python_test` instead, so
each interpreter's environment gets its own copy. Installing them from
`zopen_pre_check` does not work: that hook runs **once**, before any interpreter
is chosen, so it can only ever equip the primary.

**CRITICAL: When customizing Python test execution:**
- Keep pytest invocation verbose (`-v`) to preserve full check log output
- Ensure `zopen_check_results` parses summary text patterns like `N passed`, `N failed`, and `N errors`
- Ensure `zopen_check_results` emits nonzero `totalTests` when upstream tests exist
- Stable log formatting makes CI diagnosis and result extraction reliable
- Always provide a matching `zopen_check_results` parser for custom check flows so zopen-build records accurate totals
- Do not continue to finalization, publishing, repo creation, or CI creation unless parsed test success is greater than 90%

### Writing `zopen_check_results` correctly

Every interpreter writes to **one** check log, so a parser that reads a single
result reports a fraction of the suite and, worse, lets a passing interpreter
hide a failing one. Four mistakes, all found in shipped ports:

**1. Reading the last result instead of summing.**

```sh
# WRONG - reports one interpreter's count as the whole suite
totalTests=$(grep -oE "Ran [0-9]+ test" "${chk}" | tail -1 | grep -oE "[0-9]+")
# RIGHT
totalTests=$(grep -oE "Ran [0-9]+ test" "${chk}" | awk '{s+=$2} END {print s+0}')
```

**2. Treating any `OK` as success.**

```sh
# WRONG - one passing interpreter zeroes another's failures
if grep -qE "^OK" "${chk}"; then actualFailures=0; fi
```

Sum `failures=` and `errors=` instead.

**3. Counting unittest's expected failures as failures.** A `@expectedFailure`
test that fails as designed is a pass, and unittest reports it on the *success*
line — `OK (skipped=2056, expected failures=5)`. A bare `failures=` match picks
up the `failures=5` inside it, once per interpreter. Django reported 15 failures
for a fully passing run this way. Strip it first:

```sh
actualFailures=$(sed 's/expected failures=[0-9][0-9]*//g' "${chk}" \
  | grep -oE "(failures|errors)=[0-9]+" | cut -d= -f2 | awk '{s+=$1} END {print s+0}')
```

**4. `grep -c ... || echo 0`.** `grep -c` prints `0` **and exits 1** when nothing
matches, so the fallback appends a second zero and the arithmetic dies with
`FSUM9224 bad number "0\n0"`. This fires on **clean** runs, because a clean run
has no `FAIL:` lines. It is not a z/OS limitation — GNU grep is identical, so
adding `grep` as a dependency does not help.

```sh
# WRONG
passed=$(grep -c "^PASS:" "${chk}" 2>/dev/null || echo 0)
# RIGHT
passed=$(grep -c "^PASS:" "${chk}" 2>/dev/null); passed=${passed:-0}
```

**Also count an interpreter that produced no output.** A wheel that fails to
import kills its run before any counts are printed, which otherwise reads as
success — exactly the failure that matters most for a cross-compiled extension:

```sh
runs=$(grep -c "SMOKE_TOTAL=" "${chk}"); runs=${runs:-0}
interpreters=$(grep -c "Running tests with Python" "${chk}"); interpreters=${interpreters:-0}
[ "${runs}" -lt "${interpreters}" ] && actualFailures=$((actualFailures + interpreters - runs))
```

### Publishing

#### GitHub (pax)
Standard `zopen-publish` for pax files (same as any port):
```bash
zopen-publish -f \
  -p install/<name>.pax.Z \
  -m metadata.json \
  -g <TAG> \
  -t <github_token>
```

#### Pulp PyPI (wheel)
Publish the preserved wheel from `$ZOPEN_ROOT/install/dist/` to a Pulp PyPI repository:
```bash
zopen-publish \
  --whl <name>port/install/dist/<name>-<version>-*.whl \
  --pulp-url http://<host>:<port>/pypi/<repo>/ \
  --pulp-password <password>
```

Environment variables `PULP_URL`, `PULP_USER`, `PULP_PASSWORD` can be used instead of flags.

#### Both at once
```bash
zopen-publish -f \
  -p install/<name>.pax.Z \
  -m metadata.json \
  -g <TAG> \
  -t <github_token> \
  --whl install/dist/*.whl \
  --pulp-url http://<host>:<port>/pypi/<repo>/ \
  --pulp-password <password>
```

Consumers install from Pulp with:
```bash
pip install --index-url http://<host>:<port>/pypi/<repo>/simple/ <package>
```

### Testing Python Packages

**Python packages with built-in test runners (e.g., `python -m Crypto.SelfTest`) often work better than pytest on z/OS.** Always check for native test runners before forcing pytest, especially when encountering test class compatibility issues. If the package provides its own test runner:
1. Check the package documentation or README for test instructions
2. Define `zopen_python_test()` in `buildenv` to run the native test runner instead of pytest (not `zopen_custom_check` with `ZOPEN_CHECK`, which also replaces the loop over interpreters)
3. Update `zopen_check_results()` to parse the native test runner's output format

### Test Processing and Success Gate

After every successful `zopen-build`, inspect the newest `*_check.log` and confirm `zopen_check_results` processed the test output.

Required checks:
1. `zopen_check_results` must emit `actualFailures`, `totalTests`, and `expectedFailures`.
2. `totalTests` must be greater than zero when the upstream project ships usable tests.
3. Compute success rate as `(totalTests - actualFailures) * 100 / totalTests`.
4. Continue only when success rate is greater than 90%.
5. If success rate is 90% or lower, inspect failures, fix port/test issues, and rerun `zopen-build -v`.
6. If there are no usable upstream tests, document that fact in `patches/README.md` and get explicit user approval before continuing.

Do not improve the metric by silently deleting, skipping, or narrowing test coverage. Exclude tests only when they are clearly irrelevant, unavailable on z/OS, or require external services; document each exclusion and keep the remaining parsed test set representative.

For custom parsers, count errors as failures. If the runner reports skipped tests separately, do not count skips as failures unless they represent broken required functionality.

**Python test import shadowing**: When pytest runs from source directory with local package folder (e.g., `xxhash/`), it shadows the installed wheel in site-packages, causing `ModuleNotFoundError` for C extensions. This is expected Python behavior. Tests pass when run from outside source dir or after proper installation.

**CRITICAL: Handling C Extension Test Failures**

For Python C-extension ports, if pytest in zopen-build imports the source tree instead of the installed wheel and fails with `ModuleNotFoundError` for the extension:

1. **Before changing test logic**, inspect the newest `*_check.log` to confirm whether failures come from test execution or import-path shadowing
2. A pattern like "collected tests" followed by `ModuleNotFoundError` for `package._extension` usually means pytest is running from the source tree and not exercising the installed wheel
3. **Define `zopen_python_test`** to copy the test suite to a temporary
   directory outside the source tree and run pytest there, so imports resolve
   to the installed package. Do **not** define `zopen_custom_check` and
   override `ZOPEN_CHECK`: zopen-build already activates the right virtual
   environment and installs that interpreter's wheel before calling the hook,
   and replacing `ZOPEN_CHECK` discards the loop over interpreters as well.
4. **Provide matching `zopen_check_results`** parser to extract passed/failed/error counts from the pytest summary line

Example for ports with bundled or generated tests:
```sh
zopen_python_test() {
  # Copy tests outside the source tree, where the source package would
  # otherwise shadow the installed wheel.
  test_dir="/tmp/${ZOPEN_PROJECT_NAME}_tests_$$"
  rm -rf "${test_dir}"
  mkdir -p "${test_dir}" || return $?
  cp -R tests/* "${test_dir}/" || return $?

  # Subshell, so a failure cannot leave the build in the temp directory.
  (cd "${test_dir}" && python -m pytest -v)
  zopen_check_result=$?

  rm -rf "${test_dir}"
  return "${zopen_check_result}"
}
```

Copy `tests` as a subdirectory rather than its contents if the suite uses
CWD-relative fixtures such as `Path("tests/files/x")`, and avoid the substring
`tests` in the temp directory name if any test asserts on `os.getcwd()`.

This approach:
- Validates the packaged extension exactly as CI/install will use it
- Ensures the check phase explicitly stages tests into a temporary execution directory
- Confirms the built wheel contains the extension module
- Runs tests against the installed wheel from outside the source tree
- Validates extracted test content separately from the source checkout

**A successful local import test alone is not enough to validate CI**; always confirm the package passes through the zopen-build check phase with the installed wheel plus extracted/copied tests, because pytest collection context can differ from ad hoc manual testing.

### 2. Map Dependencies (Strict)

Dependency source of truth:
- `https://raw.githubusercontent.com/zopencommunity/meta/refs/heads/main/docs/api/zopen_releases_latest.json`

Rules:
1. Map each required brew dependency to exact zopen package name.
2. Keep required dependencies only.
3. If required dependency is unavailable in zopen package list, fail and explain.
4. **Always add `coreutils`** if the project uses `make install` and the system `install` is insufficient.

Special cases:
- use `check_python` (not `python`)
- use `check_go` (not `go`)
- if `flex` is required, add `m4` before `flex`
- if `cmake` is required, add `make`
- if configure fails with "requires GNU bison", add `bison` to `ZOPEN_STABLE_DEPS`
- if build fails with "envsubst: FSUM7351 not found", add `gettext` to `ZOPEN_STABLE_DEPS` (envsubst is provided by gettext and is commonly used in Makefiles for template variable substitution)

### 3. Generate Project

Before generating, ensure `<name>port` does not exist (or use `--force` intentionally).

Non-interactive template:

```bash
zopen-generate \
  --name <name> \
  --description "<description>" \
  --categories "<cat1 cat2>" \
  --license <spdx_or_unknown> \
  --type BUILD \
  --build-system "<GNU Make|CMake|Go|Gradle|Maven|Meson|Python>" \
  --stable-url "<url>" \
  --stable-deps "<dep1 dep2>" \
  --build-line stable \
  --dev-deps "<dep1 dep2>" \
  --non-interactive
```

Notes:
- Use `--build-system Go` for Go projects.
- Keep upstream source URLs (`--stable-url`, `--dev-url`) as `https://` URLs.
- **CRITICAL: Sanitize `buildenv` variables**: Shell variables CANNOT contain hyphens. Always use underscores (e.g., `PYTHON_XXHASH_VERSION`, not `PYTHON-XXHASH_VERSION`). Hyphenated names break shell parsing and can surface later as invalid version or loadBuildEnv failures during install/finalization.
- **For git-based ports**: Specify the HTTPS git URL (e.g., `https://github.com/org/project.git`) rather than a tarball URL unless absolutely necessary. This is appropriate for projects that require git history or submodules.
- **For CMake projects**: Always reference existing working examples like `github.com/zopencommunity/llamacppport/blob/main/buildenv` before starting.
- **Dependency Home Variables**: `zopen-build` automatically provides the root directory of each dependency as an environment variable named `<DEPNAME>_HOME` (e.g., `BLIS_HOME`, `ZOSLIB_HOME`). Reference these in `buildenv` as `\${DEPNAME_HOME}` to ensure they are evaluated correctly during the build process.

### 4. Build and Iterate

```bash
cd <name>port
zopen-build -v
```

**CRITICAL: Always run `zopen-build` in foreground with appropriate timeout (e.g., 300s for typical builds, longer for complex projects). Never use background processes.** This ensures proper error capture and allows immediate debugging of build failures.

If build fails:
1. inspect latest `log.STABLE`/`log.DEV`
2. identify root cause
3. modify source or `buildenv`
4. rerun `zopen-build -v`

If build succeeds:
1. inspect the newest `*_check.log`
2. confirm `zopen_check_results` parsed test totals
3. verify parsed test success rate is greater than 90%
4. only then proceed to finalization, publishing, repository creation, and CI/CD creation

#### Handling Patch Conflicts

When patches fail to apply cleanly (common with line ending or format issues on z/OS):

**Best Practice: Individual File Resolution**
1. **Fix conflict locally**: Manually resolve the conflict in the extracted source file. Ensure the fix is correct for z/OS.
2. **Create replacement patch**:
   ```bash
   cd <extracted-source-dir>
   git add <file>
   git diff HEAD -- <file> > ../patches/<file>.patch
   ```
3. **Reset and Reapply**:
   ```bash
   git reset --hard
   cd ..
   zopen-build -v
   ```
   This ensures that the build starts from a clean state with your corrected patch correctly applied by `zopen-build`.

**Alternative: Force Apply**
1. **Force apply patches to generate rejection files**:
```bash
zopen-build -v --forcepatchapply
```
This applies patches where possible and creates `.rej` files for rejected hunks.

2. **Manually resolve conflicts**:
   - Locate `.rej` files in the extracted source directory
   - Apply rejected changes manually to the corresponding source files
   - Use the `.rej` file content as a guide for what needs to be changed

3. **Create corrected patches**:
```bash
cd <extracted-source-dir>
git diff HEAD > ../patches/PR1.patch
```

4. **Clean and rebuild**:
```bash
cd ..
zopen-build -v --clean
zopen-build -v
```

Common fixes:
- missing configure: set `ZOPEN_BOOTSTRAP` or `ZOPEN_CONFIGURE="skip"`
- missing macros/functions: add `-D__XPLAT` in `ZOPEN_EXTRA_CPPFLAGS`, rebuild with `-f` if needed
- platform differences: guard with `#ifdef __MVS__`
- **Missing symbols in `.so`**: Add `-fvisibility=default` to `ZOPEN_EXTRA_CFLAGS` or patch headers with `__attribute__((visibility("default")))`.
- **Read-only `/usr/local` errors**: Use `ZOPEN_EXTRA_MAKE_OPTS` to override install paths (e.g., `export ZOPEN_EXTRA_MAKE_OPTS="INSTALL_LIB_DIR=\${ZOPEN_INSTALL_DIR}/lib"`).
- **Big Endian issues**: Check for bit-packing or binary format assumptions. Disable `mmap` if byte-swapping is needed in memory.
- **u_int*_t typedef conflicts**: Add `#ifndef __MVS__` guards around `u_int8_t`, `u_int16_t`, `u_int64_t` typedefs (z/OS uses standard `uint*_t` types).
- **Missing symbols in `.so` (CRITICAL for extensions)**: 
  1. Add `-fvisibility=default` to `ZOPEN_EXTRA_CFLAGS` in `buildenv`
  2. Patch header files: change `#define API_MACRO` to `#define API_MACRO __attribute__((visibility("default")))` for non-Windows platforms
  3. For template headers (`.h.tmpl`), add platform check: `#ifndef _WIN32 ... #else ... __attribute__((visibility("default"))) ... #endif`
- **Read-only `/usr/local` errors**: 
  1. Use `ZOPEN_EXTRA_MAKE_OPTS` to override install paths: `export ZOPEN_EXTRA_MAKE_OPTS="INSTALL_LIB_DIR=\${ZOPEN_INSTALL_DIR}/lib INSTALL_INCLUDE_DIR=\${ZOPEN_INSTALL_DIR}/include"`
  2. Set `ZOPEN_INSTALL_OPTS="install \${ZOPEN_EXTRA_MAKE_OPTS}"` to ensure overrides are passed to `make install`
  3. Patch Makefile to use `?=` instead of `=` for install directory variables (e.g., `INSTALL_LIB_DIR ?= /usr/local/lib`)
- **Big Endian issues**: Check for bit-packing or binary format assumptions. Disable `mmap` if byte-swapping is needed in memory.
- **posix_memalign missing declaration**: Ensure `#define _XOPEN_SOURCE 600` is at the VERY TOP of the C file, before any includes.
- **thread_local support**: z/OS Clang may not support `thread_local`. Use thread-specific storage or remove if safe.
- **poll() conflicts**: `#define __poll 1` in `poll.h` can conflict with variables named `__poll`. `#undef __poll` after including `<poll.h>` on z/OS.
- Some Python packages (e.g., psutil) check `sys.platform` but lack z/OS support. Patch the Python script to add z/OS handling similar to existing platforms like AIX. Add `elif sys.platform.startswith('zos'):` cases with appropriate z/OS configuration.


### 5. Finalize After Success

Before finalization, the test-processing gate must pass: check results parsed, `totalTests > 0` when upstream tests exist, and success rate greater than 90%.

1. Create patch from extracted source tree:
```bash
git diff HEAD > ../patches/PR1.patch
```
2. Verify package/binary:
```bash
zopen-info <name>
```
3. If exporting headers/libs for downstream ports, update `zopen_append_to_env` in `buildenv`.
4. Validate bump config:
```bash
bump --help
bump current buildenv
bump check buildenv
```
**CRITICAL for bump configuration**:
- Ensure version variable (e.g., `USEARCH_VERSION`) is defined BEFORE `ZOPEN_STABLE_URL` references it
- **Use git repository URL pattern** for GitHub projects: `https://github.com/org/project.git|*` (NOT the releases page URL `https://github.com/org/project/releases|semver:*`, which fails with "no version found" error)
- Follow existing port examples like `gitport` for correct bump pattern syntax

5. Add source-dir ignore pattern to `.gitignore`:
```bash
echo "" >> .gitignore
echo "# Ignore source directories created by zopen-build" >> .gitignore
echo "<package-name>-*/" >> .gitignore
echo "<package-name>/" >> .gitignore
```
6. **NEVER check in the extracted source directory.** Verify with `git status` before committing.
7. Document changes in `patches/README.md`.

## Repository and CI/CD

Create the zopencommunity repository and CI/CD job after the port is validated. This is part of the default completion path for Python ports.

Only skip this step if the user explicitly says not to create the repository or pipeline. If credentials or org permissions are missing, stop and report the exact command that failed plus the missing access needed.

1. Create the repository:
```bash
zopen-create-repo --help
zopen-create-repo -n <name> -d "zopen port of <name>"
```
Fallback for token issues: `unset GITHUB_TOKEN; gh repo create zopencommunity/<name>port --public --description "..."`.

2. Configure the SSH remote and push:
```bash
git remote add origin git@github.com:zopencommunity/<name>port.git
git push origin main
```

If `origin` already exists, verify it points to `git@github.com:zopencommunity/<name>port.git` before pushing. Do not overwrite an unrelated remote without asking the user.

**CRITICAL: GitHub `workflow` scope required for `.github/workflows/` files.**
Pushing files under `.github/workflows/` (including non-YAML files) requires the `workflow` scope on the PAT.
The standard `repo` scope alone is insufficient — GitHub rejects pushes with:
`refusing to allow a Personal Access Token to create or update workflow ... without 'workflow' scope`

If the PAT lacks `workflow` scope, use **per-repo deploy keys** as a workaround:
```bash
# Check token scopes
curl -sI -H "Authorization: token $GITHUB_TOKEN" https://api.github.com/user | grep x-oauth-scopes

# If 'workflow' missing: generate a deploy key per repo (each key can only be on ONE repo)
ssh-keygen -t ed25519 -f /tmp/deploy_<name>port -N "" -C "deploy-<name>port"
pubkey=$(cat /tmp/deploy_<name>port.pub)
curl -s -X POST -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/zopencommunity/<name>port/keys" \
  -d "{\"title\":\"deploy-key\",\"key\":\"${pubkey}\",\"read_only\":false}"

# Configure SSH to use that key
cat >> ~/.ssh/config <<EOF
Host github-<name>
  HostName github.com
  User git
  IdentityFile /tmp/deploy_<name>port
  StrictHostKeyChecking no
EOF

# Push via deploy key alias
git remote set-url origin git@github-<name>:zopencommunity/<name>port.git
git push origin main
```
Note: A single SSH key registered as a user-level key on github.com also works if available.
Deploy keys are per-repo; generate a fresh key for each repo.

3. Create the CI/CD job:
```bash
zopen-create-cicd-job --help
zopen-create-cicd-job -n <name> -b stable -s cicd-stable.groovy -r yes
```

**Fallback when `zopen-create-cicd-job` is unavailable** (e.g., running on LoP, not z/OS):
Call the Jenkins proxy endpoint directly:
```bash
CAUSE="Triggered for port: <name>"
ENCODED_CAUSE=$(python3 -c "import urllib.parse; print(urllib.parse.quote('$CAUSE'))")
curl -k -X POST -s -i \
  "https://cicd.zopen.community/proxy/create-job?token=zopencreatejob&cause=${ENCODED_CAUSE}" \
  -F "PORT_NAME=<name>" \
  -F "BUILD_TYPE=stable" \
  -F "SCRIPT_NAME=cicd-stable.groovy" \
  -F "RUN_JOB_AFTER=no"
# HTTP 201 = success; location header gives the queue item URL
```

4. Verify the repository and pipeline setup:
```bash
git remote -v
git status --short
```

The working tree should contain only intentional port files before push. Never push extracted source directories created by `zopen-build`.

## z/OS Shell and Environment Traps

These are not theoretical; each one shipped in a real port or pipeline.

**z/OS `find` has no `-path`.** It rejects the expression entirely
(`FSUM6372`). With `2>/dev/null` the search silently matches nothing. This made
a CI step report "produced no wheel" for a wheel it had just built.

**z/OS `grep` has no `-o`/`-oE`** (`FSUMA930`).

**GNU tools are not reliably on PATH.** zopen ships them in
`/data/zopen/usr/local/altbin`, which is on PATH on *some* build agents and not
others — so the same code passes on one agent and fails on another. Crucially:
dependency `.env`s are sourced **inside** `zopen-build`, so a declared tool
(e.g. `grep`) is available to buildenv functions; code running **outside**
`zopen-build`, such as a Jenkins shell step, gets whatever `/bin` provides.
Adding a dependency does not help there. Prefer POSIX constructs, or a shell
glob over `find`.

**`zopen-build` is `#!/bin/sh`.** No `[[ ]]`, `local`, `read -d`, or process
substitution in anything it sources.

**Unbalanced `)` inside `$( )` breaks the parser** (`FSUM7332`), even when
quoted — `$(... | sed 's/)//')` fails. Balanced parens are fine.

**File encoding.** Files may be tagged ASCII (`ISO8859-1`) or untagged EBCDIC,
and this differs per machine even for the same file. Sourcing a tagged ASCII
file from a non-interactive `ssh` shell fails with a syntax error unless
`_BPXK_AUTOCVT=ON` is set — that is not corruption, do not "fix" the file.
When rewriting one, preserve its original encoding and tag: convert with the
host's own `iconv` and restore the tag with `chtag`. Use `chtag -b` for
binaries; a wheel is a zip.

**IBM's Python version banner differs from upstream.**
`python3 --version` prints `IBM Open Enterprise SDK for Python 3.12.13`, so the
version is the **last** field, not the second. Taking field two yields `Open`
and the build fails with `The version string "Open" uses an invalid version
format`. Better still, ask Python:
`python3 -c 'import sys; print(".".join(map(str, sys.version_info[:3])))'`.

**Python 3.14 changed the platform tag** from `os390_<release>_<model>` to
`zos`, and moved from `/usr/lpp/IBM/cyp/v3rNN` to `/usr/lpp/IBM/python/v3r14`.
Anything matching `os390*` or assuming the `cyp` prefix misses 3.14.

**Compiled output is not reproducible.** The z/OS binder stamps build date and
time into the program object, so an unchanged source rebuild yields a different
`.so`. `SOURCE_DATE_EPOCH` does not reach it — it controls zip entry mtimes,
not the binder. Expect every rebuild of a compiled port to differ.

**Republishing an existing version.** `zopen-publish` refuses to replace a
published wheel. Pass `--on-conflict build-tag` and it compares the two by
content — identical contents report as already published, a real difference is
reuploaded under the next PEP 427 build tag (`pkg-1.0-1-cp312-none-any.whl`),
which pip prefers. Nothing in the index is ever overwritten.

**Do not fabricate the version.** The port template ships
`zopen_get_version() { echo "1.0.0" ... }`. Leaving it means `.version`,
`metadata.json` and the RPM name are all wrong. Echo the version variable
declared at the top of the buildenv. Equally, if a `# bump:` line updates a
version variable, make sure `ZOPEN_STABLE_TAG` uses it
(`"v${PKG_VERSION}"`) — a hardcoded tag drifts, and the port then builds one
version while reporting another.

## Completion Criteria

Port is complete when:
1. `zopen-build` succeeds.
2. the newest check log is processed by `zopen_check_results`.
3. parsed `totalTests` is greater than zero when upstream tests exist.
4. parsed test success rate is greater than 90%, or the user explicitly approved continuing with documented failures/no usable tests.
5. dependencies are exact zopen names.
6. patches are generated after success.
7. bump checks pass.
8. `.gitignore` includes source-dir pattern.
9. `patches/README.md` is updated.
10. zopencommunity repository exists and `origin` points to it.
11. port changes are pushed to the repository.
12. CI/CD job is created for the stable build line.
