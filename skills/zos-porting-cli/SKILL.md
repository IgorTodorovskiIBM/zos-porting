---
name: zos-porting-cli
description: Use this skill when porting open-source software to z/OS with zopen CLI commands. It provides a command-first workflow for metadata collection, dependency mapping to exact zopen package names, project generation, build/fix iteration, patch creation, bump validation, and optional repo/CI setup.
---

# z/OS Porting (CLI-First)

Use this skill for end-to-end zopen porting work with local `zopen-*` commands.

## Core Rules

1. Use local CLI commands directly (`zopen-generate`, `zopen-build`, `zopen-info`, etc.).
2. Use `--help` as source of truth for flags/syntax in installed tooling.
3. Prefer Homebrew formula metadata and upstream project metadata first; use web search only as fallback.
4. Do not create files in `patches/` until build succeeds.

## Preflight

Run:

```bash
command -v zopen-generate zopen-build zopen-info zopen-query zopen-version jq git bump
zopen-generate --help
zopen-build --help
zopen-info --help
zopen-version || zopen-version --help
```

If a command reports `Source the zopen-config prior to running ...`, source zopen config first.

## Workflow

### 1. Collect Metadata

Gather:
- project name (lowercase)
- one-line description
- repo URL
- SPDX license
- categories
- build system
- stable URL + dependencies

Use:
- `zopen-generate --list-licenses`
- `zopen-generate --list-categories`
- `zopen-generate --list-build-systems`
- `https://formulae.brew.sh/api/formula/${PROJECT}.json`

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
- **Sanitize `buildenv` variables**: Ensure all custom variables use underscores instead of hyphens (e.g., `SQLITE_VEC_VERSION`, not `SQLITE-VEC_VERSION`).

### 4. Build and Iterate

```bash
cd <name>port
zopen-build -v
```

If build fails:
1. inspect latest `log.STABLE`/`log.DEV`
2. identify root cause
3. modify source or `buildenv`
4. rerun `zopen-build -v`

Common fixes:
- missing configure: set `ZOPEN_BOOTSTRAP` or `ZOPEN_CONFIGURE="skip"`
- missing macros/functions: add `-D__XPLAT` in `ZOPEN_EXTRA_CPPFLAGS`, rebuild with `-f` if needed
- platform differences: guard with `#ifdef __MVS__`
- **Missing symbols in `.so`**: Add `-fvisibility=default` to `ZOPEN_EXTRA_CFLAGS` or patch headers with `__attribute__((visibility("default")))`.
- **Read-only `/usr/local` errors**: Use `ZOPEN_EXTRA_MAKE_OPTS` to override install paths (e.g., `export ZOPEN_EXTRA_MAKE_OPTS="INSTALL_LIB_DIR=\${ZOPEN_INSTALL_DIR}/lib"`).
- **Big Endian issues**: Check for bit-packing or binary format assumptions. Disable `mmap` if byte-swapping is needed in memory.

### 5. Finalize After Success

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
5. Add source-dir ignore pattern to `.gitignore`:
```bash
echo "" >> .gitignore
echo "# Ignore source directories created by zopen-build" >> .gitignore
echo "<package-name>-*/" >> .gitignore
echo "<package-name>/" >> .gitignore
```
6. **NEVER check in the extracted source directory.** Verify with `git status` before committing.
7. Document changes in `patches/README.md`.

## Optional Repo/CI

Only for users with required org permissions.

1. Ask user whether to create repo now:
```bash
zopen-create-repo --help
zopen-create-repo -n <name> -d "zopen port of <name>"
```
Fallback for token issues: `unset GITHUB_TOKEN; gh repo create zopencommunity/<name>port --public --description "..."`.

2. Push using SSH remote:
```bash
git remote add origin git@github.com:zopencommunity/<name>port.git
git push origin main
```

3. Ask user whether to create CI job now:
```bash
zopen-create-cicd-job --help
zopen-create-cicd-job -n <name> -b stable -s cicd-stable.groovy -r yes
```

## Completion Criteria

Port is complete when:
1. `zopen-build` succeeds.
2. dependencies are exact zopen names.
3. patches are generated after success.
4. bump checks pass.
5. `.gitignore` includes source-dir pattern.
6. `patches/README.md` is updated.
