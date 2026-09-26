---
name: cljd-smells-fix
description: "Fix ClojureDart code against ClojureDart-specific smells; placeholder, not yet implemented. When implemented, auto-applies mechanical and DEFECT-tier findings and reports the rest. Pass --report to disable writes."
argument-hint: "[path|all] [--report]"
allowed-tools: Bash, Read, Edit, Grep, Glob
user-invocable: true
disable-model-invocation: true
---

# ClojureDart Smells Fix (Placeholder)

This skill is a placeholder. ClojureDart code review against a curated smells catalog is planned but not yet implemented. Running `/cljd-smells-fix` today will show this notice and exit. The argument grammar and mutation contract land now so the eventual implementation has a contract to honor.

## Arguments

| Input              | Target                                                                       |
|--------------------|------------------------------------------------------------------------------|
| (no argument)      | Fix changed files only (staged + unstaged)                                   |
| `all`              | Fix the full codebase, sampling high-risk and high-traffic namespaces        |
| `path/to/dir`      | Fix files under directory                                                    |
| `path/to/file.cljd`| Fix specific file                                                            |
| `--report`         | Disable all writes; produce the report only                                  |

Examples:

```
/cljd-smells-fix
/cljd-smells-fix src/my_app
/cljd-smells-fix src/my_app/core.cljd
/cljd-smells-fix all
/cljd-smells-fix --report
```

When implemented, the skill mirrors the mutation contract of `/clj-smells-fix`: Stage 1 mechanical findings and Stage 2 `DEFECT`-tier findings within a defined safety band are auto-applied; `SMELL` and `HINT` findings remain report-only; `--report` disables all writes.

When (no argument) is invoked outside a git worktree, the eventual implementation will ask the operator what to fix rather than widening silently to `all`.

The eventual implementation will exclude vendored, generated, and dependency-locked paths from broad scopes (`.gitignore` matches plus a hardcoded floor of `node_modules/`, `vendor/`, `third_party/`, `.bundle/`, `target/`, `build/`, `dist/`, `out/`, `.shadow-cljs/`, `cljd-out/`, and the standard lock files). Naming a vendored path directly through `<path>` or `<glob>` bypasses the filter for that target.

## Status

**Not implemented.** A ClojureDart-specific smells catalog is pending. Reusing the JVM-Clojure [clj-smells catalog](https://github.com/nufuturo-ufcg/clj-smells-catalog) directly produces incorrect findings because ClojureDart's runtime, interop model, and Flutter integration differ from JVM Clojure. ClojureDart has no JVM atoms/refs/agents/`core.async` in the same form, but introduces `:watch`, `:managed`, cells, Flutter widget rebuilds, and Dart interop concerns that the JVM catalog does not cover.

## Planned Categories

When implemented, this review will cover:

- **Dynamic warnings**: `DYNAMIC WARNING: can't resolve member` and inference-failure warnings that escape `cljd-fix`.
- **Dart interop**: positional vs named-argument confusion, missing type hints causing silent boxing, Python-style method names (`.__setitem`) that look right but resolve to nothing.
- **Flutter directives**: misuse of `:watch` / `:managed` / `:bind` / `:get` / `:bg-watcher`; choosing the wrong directive for the data lifecycle.
- **Widget rebuild behavior**: unnecessary rebuilds, missed rebuilds, scope leaks across widget boundaries.
- **Async patterns**: blocking inside Flutter build phases, unawaited futures, isolate misuse.
- **Generated files**: editing files under `lib/cljd-out/`, committing generated Dart, missing `.gitignore` entries.
- **deps.edn / Flutter project config**: missing `:flutter/widget` linter, missing upstream clj-kondo hooks, mixed Clojure and ClojureDart deps.

## Output

When invoked, print this notice and exit:

```
cljd-smells-fix is not yet implemented.

A ClojureDart-specific smells catalog is in development. For now, use:
  - cljd-fix for lint, format, and compile checks
  - cljd-test for the test suite
  - clojuredart skill (auto-invoked) for idiomatic guidance

To track progress, see the clojuredart-skills README and CHANGELOG.
```

Do not run any analysis. Do not invoke clj-kondo. Do not consult the JVM clj-smells catalog. Do not write to any source file.
