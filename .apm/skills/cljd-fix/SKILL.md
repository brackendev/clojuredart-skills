---
name: cljd-fix
description: "Fix a ClojureDart project (lint, format, compile, dry); format writes by default"
argument-hint: "[lint|format|compile|dry] [--report] [all]"
user-invocable: true
disable-model-invocation: true
---

# ClojureDart Fix

Run lint, format, compile, and duplicate-form checks on a ClojureDart project. The `format` step writes by default; `lint`, `compile`, and `dry` are pure-read of source.

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

Step keywords are combinable (for example, `/cljd-fix lint compile dry`). The `--report` flag may appear in any position. When `--report` is present without an explicit step keyword, every step still runs; only the format step's behavior changes.

The step keyword `dry` is the dry4clj duplicate-form scan, not a dry-run mode. Only the literal `--report` token disables writes.

## Mutation

Only the `format` step writes source. It runs `clj -M:cljfmt fix` by default, rewriting `.cljd` files in place. With `--report`, the step runs `clj -M:cljfmt check`, which exits non-zero when files would change but does not write.

The `lint` step writes upstream clj-kondo exports into `.clj-kondo/imports/tensegritics/clojuredart/` the first time it runs (a one-time bootstrap that is idempotent and committed to version control). The `compile` step writes generated Dart under `lib/cljd-out/`, which is a build artifact, not source. Neither of these writes is affected by `--report`. The `dry` step is pure-read regardless.

This skill excludes vendored, generated, and dependency-locked paths from the file set it walks. The filter combines `.gitignore` matches and a hardcoded floor (`node_modules/`, `vendor/`, `third_party/`, `.bundle/`, `target/`, `build/`, `dist/`, `out/`, `.shadow-cljs/`, `cljd-out/`, `*.lock`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Gemfile.lock`, `Cargo.lock`, `poetry.lock`, `composer.lock`). The `lint` step's clj-kondo bootstrap write into `.clj-kondo/imports/tensegritics/clojuredart/` and the `compile` step's writes to `lib/cljd-out/` are tooling and build artifacts produced by the underlying tools themselves and are outside the source-mutation scope of the rule. Naming a vendored source path directly through `<path>` or `<glob>` bypasses the filter for that target.

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

Scan for duplicate top-level forms with [dry4clj](https://github.com/unclebob/dry4clj). This step is advisory. It reports duplication candidates for review but never fails the run and never halts the pipeline.

Determine the project's production source directories from its build configuration instead of assuming a directory name:

- `deps.edn`: the top-level `:paths` (the `:dry4clj` alias is defined here). Tests usually live under a separate alias's `:extra-paths`, so scanning `:paths` excludes them.

Scan the production source paths only and exclude test paths. If no configuration declares source paths, scan the source directories that exist in the repository, excluding any test directory. Do not assume `src`.

Run dry4clj with EDN output over the resolved paths:

```bash
clj -M:dry4clj --edn <source-paths...>
```

Do not pass `--threshold`, `--min-lines`, or `--min-nodes`. Scoping to production source removes the dominant noise, which is repeated test scaffolding; tightening the score or size knobs either hides real matches or has no effect on exact-structure duplicates.

dry4clj always exits 0, and the EDN output never prints a clean-state message, so do not use the exit code or any text match as the signal. Parse the EDN map `{:candidates [...]}` and classify the result:

| Result   | Condition                                                                                                                  |
|----------|----------------------------------------------------------------------------------------------------------------------------|
| `PASS`   | `:candidates` is empty.                                                                                                     |
| `REVIEW` | `:candidates` has one or more entries. List them and continue.                                                             |
| `ERROR`  | The command cannot run or its output cannot be parsed (for example, the `:dry4clj` alias is missing). Report the cause; do not report `PASS`. |
| `SKIP`   | No production source path can be identified.                                                                               |

For `REVIEW`, sort candidates by exact matches first (`:score` equal to `1.0`), then by descending `min(:left-nodes, :right-nodes)`, then by descending `:score`. Report the scanned source paths, the candidate count, and the highest-priority candidates with their score, node counts, and both file ranges. Note that each candidate needs source inspection before extraction, and that test directories were excluded because repeated test scaffolding is often intentional.

dry4clj's `source-extensions` set includes `.cljd`, so ClojureDart source is scanned without additional configuration, provided the project's `:dry4clj` alias points at a `unclebob/dry4clj` revision that carries the `.cljd` extension. Older revisions scan only `.clj`, `.cljc`, and `.cljs`.

## Report

After running all requested steps, print a summary:

```
ClojureDart Fix Results:
  Lint:    PASS/FAIL/SKIPPED
  Format:  PASS/FAIL/SKIPPED
  Compile: PASS/FAIL/SKIPPED
  Dry:     PASS/REVIEW/ERROR/SKIPPED
```

If `--report` was passed, append `(report mode: format checked, not written)` after the summary. If a lint, format, or compile step fails, stop and report the failure; do not continue to subsequent steps. The dry step is advisory: it never fails the run and never halts the pipeline.
