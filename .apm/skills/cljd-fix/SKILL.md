---
name: cljd-fix
description: >-
  Fix a ClojureDart project with lint, format, compile, and dry
  (duplicate-form) steps. The format step writes .cljd files by default, and
  the compile step checks the generated Dart with dart analyze. Use when asked
  to lint, format, compile, or check a ClojureDart Flutter project, or to fix
  clj-kondo warnings or cljfmt formatting in .cljd files. Use --report to
  replace the formatting write with cljfmt check.
argument-hint: "[lint|format|compile|dry] [--report] [all]"
user-invocable: true
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
| `compile`         | Run compile only, including its Dart check                                   |
| `dry`             | Run dry only                                                                 |
| `--report`        | Replace `cljfmt fix` with non-writing `cljfmt check` in the format step      |

Step keywords are combinable (for example, `/cljd-fix lint compile dry`). The `--report` flag may appear in any position. When `--report` is present without an explicit step keyword, every step still runs; only the format step's behavior changes.

The step keyword `dry` is the dry4clj duplicate-form scan, not a dry-run mode. Only the literal `--report` token disables writes.

## Mutation

Only the `format` step writes source. It runs `cljfmt fix` through the runner the Format step below selects, rewriting `.cljd` files in place. With `--report`, the step runs `cljfmt check` through the same runner, which exits non-zero when files would change but does not write.

The `lint` step writes upstream clj-kondo exports into `.clj-kondo/imports/tensegritics/clojuredart/` the first time it runs (a one-time bootstrap that is idempotent and committed to version control). The `compile` step writes generated Dart under `lib/cljd-out/`, which is a build artifact, not source, and its `pub get` can update `.dart_tool/` and `pubspec.lock`. Its Dart check reads the generated files without editing them. Neither of these writes is affected by `--report`. The `dry` step is pure-read regardless.

This skill excludes vendored, generated, and dependency-locked paths from the file set it walks. The filter combines `.gitignore` matches and a hardcoded floor (`node_modules/`, `vendor/`, `third_party/`, `.bundle/`, `target/`, `build/`, `dist/`, `out/`, `.shadow-cljs/`, `cljd-out/`, `*.lock`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Gemfile.lock`, `Cargo.lock`, `poetry.lock`, `composer.lock`). This skill takes no path argument, so the filter applies to every run. The `lint` step's clj-kondo bootstrap write into `.clj-kondo/imports/tensegritics/clojuredart/` and the `compile` step's writes to `lib/cljd-out/` are tooling and build artifacts produced by the underlying tools themselves and are outside the source-mutation scope of the rule. The Dart check names generated files under `lib/cljd-out/` because it checks the compiler's output, not because it edits them. When the filter excludes any path, add a single "skipped N vendored or generated paths" line to the report. Under `--report`, list every skipped path. The full policy is Rule 4 in the package's [CONVENTIONS.md](https://github.com/brackendev/clojuredart-skills/blob/master/CONVENTIONS.md).

## Steps

Parse `$ARGUMENTS` to determine which steps to run and whether `--report` is present. If no step keyword is supplied (or only `all` is supplied), run every step in order.

### 1. Lint

If `.clj-kondo/imports/tensegritics/clojuredart/` is missing, bootstrap the upstream exports first:

```bash
if [ ! -d .clj-kondo/imports/tensegritics/clojuredart ]; then
  mkdir -p .clj-kondo
  clj-kondo --copy-configs --dependencies --lint "$(clojure -Spath)" > /dev/null
fi
```

Run clj-kondo on both source and test trees:

```bash
clj-kondo --lint src test
```

If the project has no `test/` directory, lint `src` only.

clj-kondo copies configurations only into an existing `.clj-kondo/` directory. Without the `mkdir`, `--copy-configs` exits 0, imports nothing, and the lint reports false unresolved-symbol errors on `cljd.flutter` directives.

Report pass if exit code is 0 and there are no errors or warnings in the output. If the project's lint target is zero-errors-only, fail only on errors. Show the clj-kondo output either way.

### 2. Format

Run cljfmt with the first runner the project supports. Check each condition before running its command. `clojure -M:cljfmt fix` without a `:cljfmt` alias in `deps.edn` runs `clojure.main` on a script named `fix` and fails with `fix (No such file or directory)`.

| Runner | When | Fix | Check |
|--------|------|-----|-------|
| Project alias | `deps.edn` defines a `:cljfmt` alias under `:aliases` | `clojure -M:cljfmt fix` | `clojure -M:cljfmt check` |
| Leiningen plugin | `project.clj` lists `dev.weavejester/lein-cljfmt` in `:plugins` | `lein cljfmt fix` | `lein cljfmt check` |
| Standalone binary | `command -v cljfmt` succeeds | `cljfmt fix` | `cljfmt check` |
| Clojure CLI tool | `clojure -Ttools list` shows a `cljfmt` tool | `clojure -Tcljfmt fix` | `clojure -Tcljfmt check` |

When no runner is available, report `SKIPPED` and show the alias to add under `:aliases` in `deps.edn`:

```clojure
:cljfmt {:replace-deps {dev.weavejester/cljfmt {:mvn/version "0.16.6"}}
         :main-opts ["-m" "cljfmt.main"]}
```

`:replace-deps` keeps the project's dependencies off cljfmt's classpath, which cljfmt does not need because it reads source as text. The cljfmt README (https://github.com/weavejester/cljfmt) covers the other installation methods.

Before running cljfmt, find the configuration the selected runner reads:

- The alias, the binary, and the tool load the first of `.cljfmt.edn`, `.cljfmt.clj`, `cljfmt.edn`, and `cljfmt.clj` in the project directory, or in the nearest parent directory that has one. Accept any of these names. cljfmt ignores a `.clj` file with the warning `.clj config file '...' detected but --read-clj-config-files is not set`, because it reads that file as Clojure code. When the project's only configuration is a `.clj` file, ask the user before adding `--read-clj-config-files` to the alias or binary command. `clojure -Tcljfmt` never reads `.clj` files, so with that runner, or when the user declines, report `SKIPPED` with the reason.
- `lein cljfmt` reads the `:cljfmt` map in `project.clj`, and reads the configuration file as well only when that map sets `:load-config-file? true`.

cljfmt's default `:file-pattern`, `#"\.(?:clj[csx]?|bb)$"`, does not match `.cljd`. Without a pattern that matches `.cljd`, `cljfmt fix` and `cljfmt check` skip every ClojureDart file and still exit 0 with "All source files formatted correctly". If the configuration the runner reads has no `:file-pattern` that matches `.cljd`, report the format step as `FAIL` without running cljfmt, and show the key to add. In an `.edn` file:

```clojure
:file-pattern #re "\\.(?:clj[csxd]?|bb)$"
```

In a `.clj` file or the `project.clj` `:cljfmt` map, write the same pattern as a regular expression literal, `#"\.(?:clj[csxd]?|bb)$"`. EDN has no `#"..."` literal, and Clojure's reader has no `#re` tag.

Outside Leiningen, cljfmt formats `src`, `test`, and `project.clj` unless the configuration sets `:paths`. When the project's ClojureDart source or test directories differ, pass the existing directories as arguments: `clojure -M:cljfmt fix <paths...>` or `cljfmt fix <paths...>`, and `clojure -Tcljfmt fix :paths '["src" "test"]'` for the tool. cljfmt stops with `No such file:` when a path argument does not exist.

Without `--report`, run the runner's Fix command. With `--report`, run its Check command, which writes nothing. Report pass if exit code is 0, fail otherwise. Show any formatting changes (under `fix`) or the diff (under `check`).

### 3. Compile

Create a marker file, and note the directory it prints. The Dart check below uses it to find the files this compile writes:

```bash
marker_dir="$(mktemp -d)"
touch "$marker_dir/compile-start"
echo "$marker_dir"
```

Compile ClojureDart to Dart:

```bash
clojure -M:cljd compile
```

Report fail if exit code is non-zero, or if the output contains any `DYNAMIC WARNING: can't resolve member` lines. Those resolution failures exit 0 but indicate a method or property that does not exist on the target type, which throws `NoSuchMethodError` at runtime. Show any compilation errors and the offending warning lines.

If the command exits with `Another ClojureDart process is running (PID <pid>)`, a `clojure -M:cljd flutter` or `watch` process holds `REPL.lock` in the project. Report the step as `FAIL` with that message, and ask the user to stop the watcher. Do not delete `REPL.lock` or stop the process yourself.

#### Dart check

`clojure -M:cljd compile` writes Dart source and does not run the Dart compiler, so it passes code that `flutter build` rejects, such as a callback that returns a `Future` where Flutter expects a `void` function (Key Rule 4 in the `clojuredart` skill). When the compile passes with no `DYNAMIC WARNING` lines, run the Dart analyzer on the generated files that belong to the project. Skip the check, and report it as `SKIPPED`, when the compile failed.

1. Check that `.dart_tool/package_config.json` exists. The compile runs `flutter pub get` (`dart pub get` for a `:kind :dart` project) and continues when it fails, and `clojure -M:cljd compile --offline` skips it. Without that file, the analyzer reports every `package:` import as `uri_does_not_exist`. When the file is missing, report the check as `SKIPPED` and tell the user to run `flutter pub get`.
2. List the project's files that this compile wrote:

   ```bash
   find lib/cljd-out -name '*.dart' -newer "<marker-dir>/compile-start" -not -path 'lib/cljd-out/cljd/*'
   ```

   The compile rewrites every namespace it compiles, even without a source change, so this lists the main namespace and everything it requires. Files under `lib/cljd-out/cljd/` hold ClojureDart's own namespaces (`cljd.core`, `cljd.flutter`, `cljd.string`, and the rest), which carry analyzer warnings the project cannot fix. Files that this compile did not write are output from earlier compiles or test runs. They can be stale, and after `clojure -M:test:cljd test` some import files under `test/cljd-out/` (upstream issue #380), so leave them out. When the list is empty, report the check as `SKIPPED`.
3. Pass the listed files to the analyzer as arguments:

   ```bash
   dart analyze --no-fatal-warnings <listed files...>
   ```

   The project's `analysis_options.yaml` excludes `lib/cljd-out/**` (Key Rule 3), so `dart analyze lib/cljd-out` analyzes nothing and prints "No issues found!". The analyzer still analyzes a file named on the command line. Each generated file starts with `// ignore_for_file: type=lint, ...`, so lints stay muted while errors and warnings are reported.

Classify the result by the exit code:

| Exit | Result |
|------|--------|
| `0` | `PASS`. Show any warning lines, which do not fail the check. |
| `3` | `FAIL`. Show every `error - <file>:<line>:<column> - <message> - <code>` line. |
| any other | `FAIL`. Show the output, which names the problem (for example, a path that does not exist). |

For each error, name the source namespace with the generated file: `lib/cljd-out/my-app/screen.dart` comes from `my-app.screen` (`src/my_app/screen.cljd`). A `void` callback that returns a `Future` reports `The returned type 'Future<Object?>' isn't returnable from a 'void' function, as required by the closure's context. - return_of_invalid_type_from_closure`, which `flutter build` reports as `Can't return a value from a void function`. Fix the `.cljd` source, never the generated Dart (Iron Law CLJD-1).

A `PASS` covers the analyzer's checks on the generated Dart. It does not cover platform builds or plugin code, which only `flutter build` checks.

### 4. Dry

Scan for duplicate top-level forms with [dry4clj](https://github.com/unclebob/dry4clj). This step is advisory. It reports duplication candidates for review but never fails the run and never halts the pipeline.

Determine the project's production source directories from its build configuration instead of assuming a directory name:

- `deps.edn`: the top-level `:paths` (the `:dry4clj` alias is defined here). Tests usually live under the `:test` alias's `:extra-paths`, so scanning `:paths` excludes them. When a project keeps test namespaces under `:paths`, leave their files out by passing the production files or directories explicitly.

Scan the production source paths only and exclude test paths. If no configuration declares source paths, scan the source directories that exist in the repository, excluding any test directory. Do not assume `src`.

Check that `deps.edn` defines a `:dry4clj` alias before running the scan. When it does not, report `SKIPPED` and show the alias to add under `:aliases`:

```clojure
:dry4clj {:extra-deps {io.github.unclebob/dry4clj {:git/url "https://github.com/unclebob/dry4clj.git"
                                                    :git/sha "5994ef2bcaaae8d12bcd0afb4f091ae09083228c"}}
           :main-opts  ["-m" "dry4clj.core"]}
```

Run dry4clj with EDN output over the resolved paths:

```bash
clojure -M:dry4clj --edn <source-paths...>
```

Do not pass `--threshold`, `--min-lines`, or `--min-nodes`. Scoping to production source removes the dominant noise, which is repeated test scaffolding; tightening the score or size knobs either hides real matches or has no effect on exact-structure duplicates.

dry4clj exits 0 after a completed scan, and the EDN output never prints a clean-state message, so do not use the exit code or any text match to detect duplicates. A non-zero exit means the scan itself failed (for example, a file the reader cannot parse) and is reported as `ERROR`. Parse the EDN map `{:candidates [...]}` and classify the result:

| Result   | Condition                                                                                                                  |
|----------|----------------------------------------------------------------------------------------------------------------------------|
| `PASS`   | `:candidates` is empty.                                                                                                     |
| `REVIEW` | `:candidates` has one or more entries. List them and continue.                                                             |
| `ERROR`  | The command fails or its output cannot be parsed. Report the cause, and do not report `PASS`. |
| `SKIPPED` | No production source path can be identified, or `deps.edn` defines no `:dry4clj` alias. |

For `REVIEW`, sort candidates by exact matches first (`:score` equal to `1.0`), then by descending `min(:left-nodes, :right-nodes)`, then by descending `:score`. Report the scanned source paths, the candidate count, and the highest-priority candidates with their score, node counts, and both file ranges. Note that each candidate needs source inspection before extraction, and that test directories were excluded because repeated test scaffolding is often intentional.

The pinned dry4clj commit's `source-extensions` set includes `.cljd`, so it scans ClojureDart source without additional configuration.

dry4clj reads source without resolving namespace aliases, so an alias-qualified auto-resolved keyword such as `::str/ok` stops the scan with "Invalid token" and a non-zero exit. Plain auto-resolved keywords such as `::ok` read correctly. dry4clj also reads `#/(...)` as a plain tagged literal, so a parametrized type hint such as `^#/(List String) items` stops the scan with "Metadata must be Symbol,Keyword,String,Vector or Map" and a non-zero exit. A `#/(...)` form that is not a hint, such as the constructor call `(#/(List String))`, reads correctly. In both cases, report the file and the error as `ERROR`, keep the step advisory, and do not rewrite source to work around the reader.

## Report

After running all requested steps, print a summary:

```
ClojureDart Fix Results:
  Lint:    PASS/FAIL/SKIPPED
  Format:  PASS/FAIL/SKIPPED
  Compile: PASS/FAIL/SKIPPED
  Dart:    PASS/FAIL/SKIPPED
  Dry:     PASS/REVIEW/ERROR/SKIPPED
```

The `Dart` line is the Dart check inside the compile step. Report it as `SKIPPED` when the compile step did not run. If `--report` was passed, append `(report mode: format checked, not written)` after the summary. If a lint, format, or compile step fails, including its Dart check, stop and report the failure, and do not continue to subsequent steps.
