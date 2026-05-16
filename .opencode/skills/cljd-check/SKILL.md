---
name: cljd-check
description: Run the ClojureDart quality pipeline (lint, format, compile, dry)
argument-hint: "[lint|format|compile|dry]"
user-invocable: true
disable-model-invocation: true
---

# ClojureDart Quality Check

Run lint, format, compile, and duplicate-form checks on ClojureDart source files.

## Steps

Parse `$ARGUMENTS` to determine which steps to run. If empty, run all steps in order.

| Argument | Steps |
|----------|-------|
| (empty) | lint, format, compile, dry |
| `lint` | lint only |
| `format` | format only |
| `compile` | compile only |
| `dry` | dry only |

Multiple arguments can be combined (e.g., `lint compile dry`).

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

Run cljfmt to fix formatting:

```bash
clj -M:cljfmt fix
```

Report pass if exit code is 0, fail otherwise. Show any formatting changes.

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
ClojureDart Check Results:
  Lint:    PASS/FAIL/SKIPPED
  Format:  PASS/FAIL/SKIPPED
  Compile: PASS/FAIL/SKIPPED
  Dry:     PASS/FAIL/SKIPPED
```

If any step fails, stop and report the failure. Do not continue to subsequent steps.
