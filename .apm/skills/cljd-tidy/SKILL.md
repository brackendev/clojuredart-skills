---
name: cljd-tidy
description: Tidy a ClojureDart project (lint, format, compile, dry); format writes by default
argument-hint: "[lint|format|compile|dry] [--report] [all]"
user-invocable: true
disable-model-invocation: true
---

# ClojureDart Tidy

Run lint, format, compile, and duplicate-form checks on a ClojureDart project. The `format` step writes by default; `lint`, `compile`, and `dry` are pure-read of source. See `CONVENTIONS.md` in the repo root for the argument grammar this skill follows.

## Arguments

| Input             | Target                                                                       |
|-------------------|------------------------------------------------------------------------------|
| (no argument)     | Run all four steps (`lint`, `format`, `compile`, `dry`) on the project (src, test) |
| `all`             | Same as (no argument); accepted for family consistency                       |
| `lint`            | Run lint only                                                                |
| `format`          | Run format only                                                              |
| `compile`         | Run compile only                                                             |
| `dry`             | Run dry only                                                                 |
| `--report`        | Replace `cljfmt fix` with non-writing `cljfmt check` in the format step      |

Step keywords are combinable (for example, `/cljd-tidy lint compile dry`). The `--report` flag may appear in any position. When `--report` is present without an explicit step keyword, every step still runs; only the format step's behavior changes.

The step keyword `dry` is the dry4clj duplicate-form scan, not a dry-run mode. Only the literal `--report` token disables writes.

## Mutation

Only the `format` step writes source. It runs `clj -M:cljfmt fix` by default, rewriting `.cljd` files in place. With `--report`, the step runs `clj -M:cljfmt check`, which exits non-zero when files would change but does not write.

The `lint` step writes upstream clj-kondo exports into `.clj-kondo/imports/tensegritics/clojuredart/` the first time it runs (a one-time bootstrap that is idempotent and committed to version control). The `compile` step writes generated Dart under `lib/cljd-out/`, which is a build artifact, not source. Neither of these writes is affected by `--report`. The `dry` step is pure-read regardless.

## Steps

Parse `$ARGUMENTS` to determine which steps to run and whether `--report` is present. If no step keyword is supplied (or only `all` is supplied), run every step in order.

### 1. Lint

If `.clj-kondo/imports/tensegritics/clojuredart/` is missing, bootstrap the upstream exports first:

```bash
if [ ! -d .clj-kondo/imports/tensegritics/clojuredart ]; then
  clj-kondo --copy-configs --dependencies --lint "$(clj -Spath)" > /dev/null
fi
```

Run clj-kondo on both source and test trees:

```bash
clj-kondo --lint src test
```

If the project has no `test/` directory, lint `src` only.

Report pass if exit code is 0 and there are no errors or warnings in the output. If the project's lint target is zero-errors-only, fail only on errors. Show the clj-kondo output either way.

### 2. Format

Without `--report`, run cljfmt to fix formatting:

```bash
clj -M:cljfmt fix
```

With `--report`, run cljfmt in check mode (no writes):

```bash
clj -M:cljfmt check
```

Report pass if exit code is 0, fail otherwise. Show any formatting changes (under `fix`) or the diff (under `check`).

### 3. Compile

Compile ClojureDart to Dart:

```bash
clj -M:cljd compile
```

Report fail if exit code is non-zero, or if the output contains any `DYNAMIC WARNING: can't resolve member` lines. Those resolution failures exit 0 but indicate a method or property that does not exist on the target type, which throws `NoSuchMethodError` at runtime. Show any compilation errors and the offending warning lines.

### 4. Dry

Scan for duplicate top-level forms with [dry4clj](https://github.com/unclebob/dry4clj):

```bash
clj -M:dry4clj src test
```

If the project has no `test/` directory, scan `src` only.

dry4clj exits 0 whether or not it finds candidates, so the step must inspect output. Report pass only when exit code is 0 and stdout contains the literal `No duplicate candidates found.`. Otherwise report fail and show the reported candidates. The project's `deps.edn` must define a `:dry4clj` alias; if the alias is missing, the Clojure CLI exits non-zero and the step fails.

Upstream dry4clj scans `.clj`, `.cljc`, and `.cljs` files only, so a stock build will skip every `.cljd` file in the project. To cover `.cljd`, the project must depend on a build that adds `.cljd` to `dry4clj.core/source-extensions` (for example the `add-cljd-extension` branch of [brackendev/dry4clj](https://github.com/brackendev/dry4clj/tree/add-cljd-extension), tracked in upstream PR [#1](https://github.com/unclebob/dry4clj/pull/1)).

## Report

After running all requested steps, print a summary:

```
ClojureDart Tidy Results:
  Lint:    PASS/FAIL/SKIPPED
  Format:  PASS/FAIL/SKIPPED
  Compile: PASS/FAIL/SKIPPED
  Dry:     PASS/FAIL/SKIPPED
```

If `--report` was passed, append `(report mode: format checked, not written)` after the summary. If any step fails, stop and report the failure. Do not continue to subsequent steps.
