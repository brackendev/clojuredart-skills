---
name: cljd-smells-review
description: Review ClojureDart code against ClojureDart-specific smells (placeholder, see TODO)
argument-hint: "[scope or options...]"
allowed-tools: Bash, Read, Grep, Glob
user-invocable: true
disable-model-invocation: true
---

# ClojureDart Smells Review (Placeholder)

This skill is a placeholder. ClojureDart code review against a curated smells catalog is planned but not yet implemented. Running `/cljd-smells-review` today will show this notice and exit.

## Status

**Not implemented.** A ClojureDart-specific smells catalog is pending. Reusing the JVM-Clojure [clj-smells catalog](https://github.com/nufuturo-ufcg/clj-smells-catalog) directly produces incorrect findings because ClojureDart's runtime, interop model, and Flutter integration differ from JVM Clojure. ClojureDart has no JVM atoms/refs/agents/`core.async` in the same form, but introduces `:watch`, `:managed`, cells, Flutter widget rebuilds, and Dart interop concerns that the JVM catalog does not cover.

## Planned Categories

When implemented, this review will cover:

- **Dynamic warnings**: `DYNAMIC WARNING: can't resolve member` and inference-failure warnings that escape `cljd-check`.
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
  - cljd-check for lint, format, and compile checks
  - cljd-test for the test suite
  - clojuredart skill (auto-invoked) for idiomatic guidance

To track progress, see TODO.md in the clojure-skills repo.
```

Do not run any analysis. Do not invoke clj-kondo. Do not consult the JVM clj-smells catalog.
