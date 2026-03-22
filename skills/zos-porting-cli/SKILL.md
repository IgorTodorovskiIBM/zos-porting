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
5. **Always run `zopen-build` in foreground with appropriate timeout, never as background process.** This ensures proper error capture and debugging.

## CRITICAL: Continuous Skill Improvement

**If the user provides a course correction, tip, or alternative approach that successfully resolves an issue, you MUST update this `SKILL.md` file in the `https://github.com/IgorTodorovskiIBM/zos-porting` repository.** This ensures the skill evolves and prevents future agents from repeating the same mistakes. Treat this as a core mandate of the porting workflow.

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

## MCP Integration (Model Context Protocol)

### Overview
This skill now integrates with the z/OS Porting Assistant MCP server, providing Bob with direct access to 600+ z/OS porting patches and documentation through structured API calls.

### Available MCP Tools

#### 1. search_patches
Search the z/OS porting patch database semantically.

**Parameters:**
- `query` (required): Search query describing the problem
- `tool_filter` (optional): Filter by tool name (e.g., "git", "vim", "bash")
- `category_filter` (optional): Filter by category ("compilation", "linking", "runtime", "configuration")
- `top_k` (optional): Number of results (default: 5)

**Example:**
```
Use search_patches to find patches about "EBCDIC conversion errors" for git, limit 3 results
```

#### 2. analyze_error
Classify and analyze build/runtime errors.

**Parameters:**
- `error_message` (required): The error message to analyze
- `context` (optional): Additional context (file path, command, etc.)

**Example:**
```
Use analyze_error to classify: "undefined reference to `__etoa_l`"
```

#### 3. set_build_context
Set build context for more relevant search results.

**Parameters:**
- `tool_name` (optional): Tool being ported
- `category` (optional): Current phase (compilation/linking/runtime)
- `error_patterns` (optional): List of error patterns

**Example:**
```
Use set_build_context with tool "git" and category "compilation"
```

#### 4. get_build_context
View current build context.

**Example:**
```
Use get_build_context to see what context is set
```

#### 5. clear_build_context
Reset build context.

**Example:**
```
Use clear_build_context to reset the context
```

#### 6. get_related_patches
Find patches commonly applied together.

**Parameters:**
- `patch_filename` (required): Reference patch filename

**Example:**
```
Use get_related_patches for "git_configure.patch"
```

#### 7. search_documentation
Search z/OS API documentation.

**Parameters:**
- `query` (required): Search query
- `top_k` (optional): Number of results (default: 5)

**Example:**
```
Use search_documentation to find info about "EBCDIC conversion functions"
```

### Error → MCP Tool Mapping

When Bob encounters build/runtime failures, use this mapping:

| Error Type | MCP Tool to Use | Example Query |
|------------|----------------|---------------|
| Compilation error | `analyze_error` → `search_patches` | "implicit declaration of function" |
| Linking error | `analyze_error` → `search_patches` | "undefined reference to symbol" |
| Runtime crash | `analyze_error` → `search_patches` | "segmentation fault in function" |
| Configuration issue | `search_patches` | "configure script fails on z/OS" |
| Unknown error | `analyze_error` first | Full error message |

### Workflow Example

**Scenario:** Git compilation fails with EBCDIC error

```
1. Bob encounters error: "implicit declaration of function '__etoa_l'"

2. Bob uses analyze_error:
   - Classifies as: compilation error
   - Category: EBCDIC conversion
   - Suggests: search for EBCDIC patches

3. Bob uses search_patches:
   - Query: "EBCDIC conversion __etoa_l"
   - Tool filter: "git"
   - Category: "compilation"
   - Finds: git_ebcdic_conversion.patch

4. Bob uses get_related_patches:
   - Input: "git_ebcdic_conversion.patch"
   - Finds: Often applied with "git_configure.patch"

5. Bob provides complete solution:
   - Apply both patches
   - Rebuild with correct flags
   - Verify fix
```

### Setup Requirements

The MCP server must be configured in Bob's settings:

**Location:** `~/Library/Application Support/Code/User/mcp.json`

**Configuration:**
```json
{
  "servers": {
    "zos-porting": {
      "command": "/path/to/zos-porting-rag/mcp_server/run_server.sh",
      "type": "stdio",
      "env": {
        "DB_HOST": "/tmp",
        "DB_PORT": "5432",
        "DB_NAME": "zopen_porting",
        "DB_USER": "your_username",
        "DB_PASSWORD": ""
      }
    }
  }
}
```

### Benefits

- **Semantic Search:** Find patches by describing the problem, not just keywords
- **Context-Aware:** Results improve as Bob learns about your build environment
- **Relationship Discovery:** Find commonly co-applied patches
- **Error Classification:** Automatic categorization of build/runtime failures
- **Documentation Access:** Query z/OS API docs directly
- **Structured Data:** JSON responses for reliable parsing

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

Special cases:
- use `check_python` (not `python`)
- use `check_go` (not `go`)
- if `flex` is required, add `m4` before `flex`
- if `cmake` is required, add `make`
- if configure fails with "requires GNU bison", add `bison` to `ZOPEN_STABLE_DEPS`

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
- **CRITICAL: `ZOPEN_STABLE_URL` must point to actual release tarball** (e.g., `https://github.com/org/project/releases/download/vX.Y.Z/project-X.Y.Z.tar.gz`), **NOT GitHub API endpoints** (e.g., `https://api.github.com/repos/org/project/tarball/vX.Y.Z`). API endpoints return different archive formats that may cause extraction issues.
- **CRITICAL: Sanitize `buildenv` variables**: Shell variables CANNOT contain hyphens. Always use underscores (e.g., `SQLITE_VEC_VERSION`, not `SQLITE-VEC_VERSION`). This will cause immediate build failures.

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
```
6. Document changes in `patches/README.md`.

## Optional Repo/CI

Only for users with required org permissions.

1. Ask user whether to create repo now:
```bash
zopen-create-repo --help
zopen-create-repo -n <name> -d "zopen port of <name>"
```
Requires `gh` and `GITHUB_TOKEN`/`--github-token`.

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