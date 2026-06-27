# CLAUDE.md

Guidance for Claude / AI coding agents working in this repository.

> A companion file, [`AGENTS.md`](./AGENTS.md), contains the same core guidance
> in condensed form. Keep the two in sync when either is updated. The
> human-facing source overview is in [`README.md`](./README.md).

## Communication

- **Always reply to the user in Traditional Chinese (繁體中文).** This applies to
  all chat responses and explanations. Code, identifiers, file contents, commit
  messages, and other repository artifacts stay in their original language
  (typically English).

## Usage scope

- **Education use only.** Claude Code is used here strictly for *learning and
  understanding* this codebase, **not** for production work. Favor explanation,
  walkthroughs, and exploration of how the existing code works. Do not treat
  edits as production deliverables, and keep in mind the project does not accept
  agentic code upstream (see below).

## What this project is

SQLite is a self-contained, serverless, zero-configuration, transactional SQL
database engine written in C. This repository is the **complete source** for
the engine (current version is in [`VERSION`](./VERSION)). It builds the core
library, the `sqlite3` CLI shell, and a large TCL-based test suite.

### Critical, non-obvious facts

- **Public domain.** The code carries no copyright and no license header. Never
  add a copyright/license header to any file. The "blessing" comment at the top
  of each source file is intentional — preserve it unchanged.
- **Git is a read-only mirror.** The canonical VCS is [Fossil](https://fossil-scm.org/),
  repo at <https://sqlite.org/src>. Git check-in names differ from official
  Fossil names; the official check-in hash is in [`manifest.uuid`](./manifest.uuid).
  Use the official (Fossil) name when referring to a check-in.
- **Pull requests are generally not accepted** (a PR could carry a copyright and
  remove the code from the public domain). The project does **not** accept
  agentic code, but **does** welcome agentic *bug reports with a reproducible
  test case*, and PRs/patches as proof-of-concept for documentation purposes.

## Build

The `./configure` script uses [autosetup](https://msteveb.github.io/autosetup/),
**not** GNU Autoconf. Out-of-tree builds (a separate build directory) are
recommended but not required.

```bash
apt install gcc make tcl-dev      # prerequisites (Debian/Ubuntu); TCL 8.6+

./configure --dev                 # debug build (developer settings)
# or: ../sqlite/configure --all --debug CFLAGS='-O0 -g'   # out-of-tree debug

make sqlite3                      # the CLI shell (sqlite3)
make sqlite3d                     # debugging variant of the CLI shell
make sqlite3.c                    # the amalgamation (single-file distribution)
make testfixture                  # the test-runner binary (needs tcl-dev)
make tclextension-install         # install the SQLite TCL extension (do before tests)
make sqldiff                      # diff tool
make sqlite3_analyzer             # space-analysis tool
```

- Core deliverables (`sqlite3.c`, `sqlite3`) build without TCL; most other
  targets and all tests need a `tclsh` 8.6+.
- Add extra compile-time flags with `OPTIONS=...`, e.g.
  `make OPTIONS=-DSQLITE_OMIT_DEPRECATED sqlite3`.
- Windows: build with MSVC via `nmake /f Makefile.msc` (or the `make.bat`
  wrapper). See [`doc/compile-for-windows.md`](./doc/compile-for-windows.md);
  Unix details in [`doc/compile-for-unix.md`](./doc/compile-for-unix.md).

## Testing

Tests are TCL scripts (`test/*.test`, ~1190 files) run through `testfixture`,
an augmented TCL interpreter, and orchestrated by `test/testrunner.tcl`.

```bash
# From the build directory:
make testfixture                  # build the runner once
./testfixture test/main.test      # run a single test file

test/testrunner.tcl               # quick/default suite
test/testrunner.tcl full          # full suite
test/testrunner.tcl fts5%         # glob/pattern match a subset

# make-driven entry points:
make devtest                      # representative dev subset — run after any src/ change
make test                         # alias for devtest
make releasetest                  # full release suite (needs valgrind, etc.)
make smoketest                    # quick sanity subset
make fuzztest                     # fuzzcheck + sessionfuzz

grep '!' testrunner.log           # scan for failures after a run
```

**Always run at least `make devtest` after any change under `src/`.** See
[`doc/testrunner.md`](./doc/testrunner.md) for the test harness in detail.

## Architecture

SQL processing pipeline (each stage maps to source files in `src/`):

```
SQL text
  → tokenizer        tokenize.c
  → parser           parse.y  (Lemon → parse.c)
  → code generator   build.c, select.c, insert.c, update.c, delete.c, expr.c
  → query optimizer  where.c / where*.c
  → VDBE bytecode    vdbe.c   (the virtual machine that runs prepared statements)
  → B-Tree           btree.c  (storage engine)
  → Pager            pager.c  (transactions, page cache)
  → WAL              wal.c
  → VFS / OS         os_unix.c, os_win.c  (pluggable OS interface)
```

Headers: the master internal header is [`src/sqliteInt.h`](./src/sqliteInt.h).
Major subsystems have private internal headers: `vdbeInt.h`, `btreeInt.h`,
`whereInt.h`. The **public API** is defined in the template
[`src/sqlite.h.in`](./src/sqlite.h.in), which generates `sqlite3.h`.

See <https://sqlite.org/arch.html> for the canonical architectural overview.

## Source tree map

| Path | Contents |
|---|---|
| `src/` | Core C source (~154 files). Files named `test*.c` are **test code**, not core; `tclsqlite.c` / `tclsqlite3.c` are the TCL bindings; `shell.c.in` builds the CLI. |
| `test/` | TCL test scripts (`*.test`), plus C test modules and harness (`testrunner.tcl`). |
| `tool/` | Build/codegen scripts and helper programs, including the Lemon parser generator (`lemon.c`) and the `mk*.tcl`/`mk*.c` generators. |
| `ext/` | Extensions, built separately from the core (some are folded into the amalgamation). See below. |
| `doc/` | Documentation of internals (build, testrunner, JSON, pager invariants, etc.). End-user/API docs live in a *separate* repository. |
| `autosetup/`, `auto.def` | The autosetup configure system. |
| `main.mk`, `Makefile.in`, `Makefile.msc` | Build rules (Unix amalgamated, autosetup, MSVC). |

### Extensions (`ext/`)

Compiled separately from the core; not all are in the amalgamation by default.

- `ext/fts5/` — Full-Text Search 5 (current FTS engine)
- `ext/fts3/` — Full-Text Search 3/4 (legacy)
- `ext/rtree/` — R-Tree spatial index
- `ext/session/` — changesets and sessions
- `ext/recover/` — database file recovery
- `ext/rbu/` — Resumable Bulk Update
- `ext/icu/` — ICU integration
- `ext/intck/` — integrity check
- `ext/expert/` — index recommendation ("expert") tool
- `ext/qrf/` — Query Result Formatter library
- `ext/jni/`, `ext/wasm/` — JNI and WebAssembly bindings
- `ext/misc/` — assorted single-file extensions (many bundled in the CLI)

## Do NOT edit generated files

These are produced by scripts and must be regenerated, never hand-edited:

| Generated file | Source / regenerate with |
|---|---|
| `sqlite3.h` | `src/sqlite.h.in` + `VERSION` + `manifest.uuid` via `tool/mksqlite3h.tcl` |
| `parse.c`, `parse.h` | `src/parse.y` via the Lemon generator (`tool/lemon.c`, template `tool/lempar.c`) |
| `opcodes.h` | scanned from `src/vdbe.c` by `tool/mkopcodeh.tcl` |
| `opcodes.c` | scanned from `opcodes.h` by `tool/mkopcodec.tcl` |
| `keywordhash.h` | `tool/mkkeywordhash.c` |
| `pragma.h` | `tool/mkpragmatab.tcl` |
| `sqlite3.c` | the amalgamation, via `tool/mksqlite3c.tcl` (after `make target_source`) |

Common change recipes:
- **Add a PRAGMA** → edit `tool/mkpragmatab.tcl`, then regenerate `pragma.h`.
- **Add a VDBE opcode** → add the `case OP_Xxx:` handler in `src/vdbe.c`; the
  opcode number/name are extracted automatically by `mkopcodeh.tcl`.
- **Change the SQL grammar** → edit `src/parse.y`, never `parse.c`.

## Coding conventions

- **C only**, C89/C99 compatible. No C++, no STL, no exceptions, no VLAs.
- **All allocation** goes through `sqlite3Malloc` / `sqlite3_malloc64` (never
  raw `malloc`). Use `sqlite3MallocZero` to zero-initialize.
- **Integer widths**: use `i64` (`sqlite3_int64`) for 64-bit, `u32`/`u64` for
  unsigned. Avoid bare `long`/`int` for values that can exceed 2 GiB.
- **Error handling**: return `SQLITE_OK` (0) on success, a `SQLITE_*` code on
  failure. Many routines also set `db->mallocFailed` on OOM for deferred
  checking.
- **Assert liberally** for invariants. Use `ALWAYS(x)` / `NEVER(x)` for
  conditions that are logically always true/false but the compiler can't prove.
- Match the surrounding style: indentation, brace placement, and the terse
  comment idiom already present in the file you are editing.

## Git workflow in this mirror

- Develop on the designated feature branch; commit with clear messages; push
  with `git push -u origin <branch>`.
- Do **not** open a pull request unless explicitly asked (see public-domain
  note above — PRs are generally not accepted upstream).
- Remember the canonical history lives in Fossil; this Git remote is a mirror.

## Useful references

- Architecture: <https://sqlite.org/arch.html>
- File format: <https://sqlite.org/fileformat2.html>
- VDBE opcodes: <https://sqlite.org/opcode.html>
- Query planner overview: <https://sqlite.org/optoverview.html>
- Atomic commit / transactions: <https://sqlite.org/atomiccommit.html>
- Compile-time options: <https://sqlite.org/compile.html>
- Lemon parser generator: <https://sqlite.org/doc/trunk/doc/lemon.html>
- Testing the TCL extension: [`doc/tcl-extension-testing.md`](./doc/tcl-extension-testing.md)
