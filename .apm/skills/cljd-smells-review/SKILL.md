---
name: cljd-smells-review
description: Review ClojureDart code against ClojureDart-specific smells; pure report, never writes (placeholder, see TODO)
argument-hint: "[path|all]"
allowed-tools: Bash, Read, Grep, Glob
user-invocable: true
disable-model-invocation: true
---

# ClojureDart Smells Review (Placeholder)

This skill is a placeholder. ClojureDart code review against a curated smells catalog is planned but not yet implemented. Running `/cljd-smells-review` today will show this notice and exit. The argument grammar and pure-report classification land now so the eventual implementation has a contract to honor. See `CONVENTIONS.md` in the repo root for the standard.

## Arguments

| Input              | Target                                                                       |
|--------------------|------------------------------------------------------------------------------|
| (no argument)      | Review changed files only (staged + unstaged)                                |
| `all`              | Review the full codebase, sampling high-risk and high-traffic namespaces     |
| `path/to/dir`      | Review files under directory                                                 |
| `path/to/file.cljd`| Review specific file                                                         |

Examples:

```
/cljd-smells-review
/cljd-smells-review src/my_app
/cljd-smells-review src/my_app/core.cljd
/cljd-smells-review all
```

This skill is pure-report: it never writes. Operators apply suggestions themselves. No `--report` flag, because there is nothing to invert.

When (no argument) is invoked outside a git worktree, the eventual implementation will ask the operator what to review rather than widening silently to `all`.

## Status

**Not implemented.** A ClojureDart-specific smells catalog is pending. Reusing the JVM-Clojure [clj-smells catalog](https://github.com/nufuturo-ufcg/clj-smells-catalog) directly produces incorrect findings because ClojureDart's runtime, interop model, and Flutter integration differ from JVM Clojure. ClojureDart has no JVM atoms/refs/agents/`core.async` in the same form, but introduces `:watch`, `:managed`, cells, Flutter widget rebuilds, and Dart interop concerns that the JVM catalog does not cover.

## Planned Categories

When implemented, this review will cover:

- **Dynamic warnings**: `DYNAMIC WARNING: can't resolve member` and inference-failure warnings that escape `cljd-tidy`.
- **Dart interop**: positional vs named-argument confusion, missing type hints causing silent boxing, Python-style method names (`.__setitem`) that look right but resolve to nothing.
- **Flutter directives**: misuse of `:watch` / `:managed` / `:bind` / `:get` / `:bg-watcher`; choosing the wrong directive for the data lifecycle.
- **Widget rebuild behavior**: unnecessary rebuilds, missed rebuilds, scope leaks across widget boundaries.
- **Async patterns**: blocking inside Flutter build phases, unawaited futures, isolate misuse.
- **Generated files**: editing files under `lib/cljd-out/`, committing generated Dart, missing `.gitignore` entries.
- **deps.edn / Flutter project config**: missing `:flutter/widget` linter, missing upstream clj-kondo hooks, mixed Clojure and ClojureDart deps.

## Tracking

See `TODO.md` in the repo root.

## Output

When invoked, print this notice and exit:

```
cljd-smells-review is not yet implemented.

A ClojureDart-specific smells catalog is in development. For now, use:
  - cljd-tidy for lint, format, and compile checks
  - cljd-test for the test suite
  - clojuredart skill (auto-invoked) for idiomatic guidance

To track progress, see TODO.md in the clojure-skills repo.
```

Do not run any analysis. Do not invoke clj-kondo. Do not consult the JVM clj-smells catalog.
