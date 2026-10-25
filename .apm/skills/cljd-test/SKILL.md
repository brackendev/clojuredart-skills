---
name: cljd-test
description: >-
  Run ClojureDart tests with cljd.test, and offer to scaffold test files when
  none exist. Use when asked to run unit tests, widget tests, or a specific
  test path in a ClojureDart Flutter project.
argument-hint: "[unit|widget|all|<path>]"
user-invocable: true
---

# ClojureDart Test

Run tests for ClojureDart projects using `cljd.test`. When no test files exist, the skill offers to scaffold them.

## Arguments

| Input             | Target                                                                       |
|-------------------|------------------------------------------------------------------------------|
| (no argument)     | Run all tests with the project's test command (`clojure -M:test:cljd test`, see Test Command) |
| `all`             | Same as (no argument); accepted for family consistency                       |
| `<path>` `<glob>` | Restrict to those compiled Dart test files (for example, `test/cljd-out/my-app/core-test_test.dart`) |
| `unit`            | Run unit-tagged tests only (`clojure -M:test:cljd test -- --tags unit`)      |
| `widget`          | Run widget-tagged tests only (`clojure -M:test:cljd test -- --tags widget`)  |

The tag keywords assume tests declare the `:tags` option (see Test Tags below). If the project does not use tags, `unit` runs all tests excluding widget test files, and `widget` runs only test files that require `flutter_test`.

This skill omits `--report` because preview is meaningless for a test run; the operator can inspect the test files and tags before invoking.

## Mutation

Running tests does not write source. The test runner compiles each test namespace to `test/cljd-out/` (or to the directory that `:dart.test/dir` names, such as `integration_test/cljd-out/`) and the namespaces it requires to `lib/cljd-out/`. Those are build-artifact writes, not source mutation. `clojure -M:cljd test` also adds the `test` package to `dev_dependencies` in `pubspec.yaml` when it is missing. When no test files exist, the skill offers to scaffold them (see Scaffold Tests below), and it writes the test files and the `:test` alias only after operator confirmation.

## Test Command

`clojure -M:cljd test` compiles every `.cljd` and `.cljc` namespace in the classpath roots that lie inside the project directory, then runs `flutter test` (`dart test` for a `:kind :dart` project) with the arguments after `--`. A directory is a classpath root only when `deps.edn` puts it there: the top-level `:paths`, or the `:extra-paths` of an alias named on the command line. Upstream `doc/TESTING.md` uses an alias for tests in an extra directory, as in `clojure -M:test:cljd test`.

Before running, find the test namespaces: the `.cljd` and `.cljc` files that require `cljd.test` and call `deftest`. Run them where they are, and do not move them between directories:

| Where the tests are | Command |
|---------------------|---------|
| `deps.edn` defines a `:test` alias, such as `:test {:extra-paths ["test"]}` | `clojure -M:test:cljd test`. It also compiles test namespaces under `:paths`. |
| No `:test` alias, and the tests are under `:paths`, such as next to the source in `src/` | `clojure -M:cljd test` |
| A directory that another alias adds, such as upstream's `:test-widgets` | `clojure -M:<alias>:cljd test` |
| A directory that no alias adds, such as `test/` without a `:test` alias | Do not run the suite. Report it, and offer to add `:test {:extra-paths ["test"]}` (see deps.edn Test Configuration). |

The commands below use `clojure -M:test:cljd test`. Use the command from the table when it differs.

## Run Tests

Without a file argument, `flutter test` runs every test file under `test/`, including compiled files left by earlier runs. Naming namespaces after `test` limits what is compiled, not what runs, so pass the compiled file too:

```bash
# Run all tests
clojure -M:test:cljd test

# Run one test namespace
clojure -M:test:cljd test my-app.core-test -- test/cljd-out/my-app/core-test_test.dart

# Run a specific compiled test file
clojure -M:test:cljd test -- test/cljd-out/my-app/core-test_test.dart

# Run with verbose output
clojure -M:test:cljd test -- --reporter expanded

# Exclude slow tests
clojure -M:test:cljd test -- --exclude-tags slow
```

A compiled test file is named after its namespace with `_test.dart` appended: `my-app.core-test` compiles to `test/cljd-out/my-app/core-test_test.dart`.

Report results:

```
ClojureDart Test Results:
  Status: PASS/FAIL
  Output: <test output>
```

If no test files exist, offer to scaffold them (see below).

## Scaffold Tests

When scaffolding, create test files under `test/` that mirror the source path, and write the following in the same confirmed change:

- When `deps.edn` has no `:test` alias, add `:test {:extra-paths ["test"]}` under `:aliases`. Without it, `clojure -M:test:cljd test` does not compile the new files.
- When `.cljfmt.edn` sets `:paths` without `"test"`, add `"test"` so that `/cljd-fix` formats the tests.

### Unit Test

For a source file `src/my_app/utils.cljd`, create `test/my_app/utils_test.cljd`:

```clojure
(ns my-app.utils-test
  (:require
   [cljd.test :refer [deftest is testing are]]
   [my-app.utils :as utils]))

(deftest test-example
  (testing "describe what is being tested"
    (is (= expected (utils/my-function input)))))
```

Test namespaces compile to `test/cljd-out/` by default (for example, `test/cljd-out/my-app/utils-test_test.dart`), where `flutter test` picks them up.

### Widget Test

Widget tests require the `flutter_test` runner. Create `test/my_app/widgets_test.cljd`:

```clojure
(ns my-app.widgets-test
  (:require
   [cljd.test :refer [deftest is testing]]
   ["package:flutter/material.dart" :as m]
   ["package:flutter_test/flutter_test.dart" :as ft]))

(deftest test-widget-renders
  :tags [:widget]
  :runner (ft/testWidgets [tester])
  (let [^ft/WidgetTester tester tester]
    (await
     (.pumpWidget tester
       (m/MaterialApp
         .home (m/Scaffold
                 .body (m/Text "Hello")))))
    (ft/expect (ft/find.text "Hello") ft/findsOneWidget)
    (ft/expect (ft/find.text "Missing") ft/findsNothing)))
```

Key differences from unit tests:
- Declare the runner with the inline option `:runner (ft/testWidgets [tester])` after the test name. `deftest` reads `:runner` and `:tags` only as inline options and ignores `^{:runner ...}` and `^{:tags ...}` metadata. With runner metadata, the body runs under the plain `test` runner and has no `tester`, so a body that uses `tester` fails to compile with `Unknown symbol: tester`. With tag metadata, tag selection matches nothing.
- The runner binds a `WidgetTester` to the name in the binding vector. Type-hint it as `^ft/WidgetTester` in the body.
- `await` every async tester method, for example `(await (.pumpWidget tester widget))`, `(await (.tap tester finder))`, and `(await (.pumpAndSettle tester))`. The body must await at least one of them: `testWidgets` expects a callback that returns a `Future`, and a body with no `await` fails with `type '...' is not a subtype of type 'Future<void>'`.
- Locate widgets with `ft/find.text`, `ft/find.byType`, and `ft/find.byKey`. `ft/find` is a top-level constant (a `CommonFinders` instance), not a class.
- Assert on finders with `(ft/expect finder ft/findsOneWidget)` or `ft/findsNothing`. A finder is never nil, so `(is (some? finder))` always passes.

### Integration Test

To compile tests into the `integration_test/` directory instead of `test/`, put `:dart.test/dir` in the namespace attribute map after the name. Metadata on the namespace symbol is ignored. Keep the source under `test/` with the other tests, for example `test/my_app/integration_test.cljd`:

```clojure
(ns my-app.integration-test
  {:dart.test/dir "integration_test"}
  (:require
   [cljd.test :refer [deftest is]]
   ["package:flutter/material.dart" :as m]
   ["package:flutter_test/flutter_test.dart" :as ft]
   ["package:integration_test/integration_test.dart" :as it]))

(deftest test-full-flow
  :runner (ft/testWidgets [tester])
  (it/IntegrationTestWidgetsFlutterBinding.ensureInitialized)
  (let [^ft/WidgetTester tester tester]
    (await (.pumpWidget tester (m/Placeholder)))
    ;; ... full app test
    ))
```

This namespace compiles to `integration_test/cljd-out/my-app/integration-test_test.dart`. A plain `clojure -M:test:cljd test` compiles it but runs only the files under `test/`. Run integration tests with:

```bash
clojure -M:test:cljd test -- integration_test
```

`clojure -M:cljd init` adds only `lib/cljd-out/` and `test/cljd-out/` (with `.cpcache/` and `.clojuredart/`) to `.gitignore`, and `clojure -M:cljd clean` deletes only `lib/cljd-out/` and `test/cljd-out/`. When the project has integration tests, add `integration_test/cljd-out/` to `.gitignore`, and delete that directory by hand when a clean build is needed.

## Test Tags

Use the inline `:tags` option to categorize tests:

```clojure
(deftest test-pure-logic
  :tags [:unit :fast]
  (is (= 4 (+ 2 2))))

(deftest test-complex-widget
  :tags [:widget :slow]
  :runner (ft/testWidgets [tester])
  (let [^ft/WidgetTester tester tester]
    (await (.pumpWidget tester (m/Placeholder)))
    ;; ...
    ))
```

Run tagged subsets:

```bash
clojure -M:test:cljd test -- --tags unit
clojure -M:test:cljd test -- --exclude-tags slow
```

## deps.edn Test Configuration

Put tests in `test/`, and add the directory with a `:test` alias:

```clojure
:aliases {:cljd {:main-opts ["-m" "cljd.build"]}
          :test {:extra-paths ["test"]}}
```

Add default test arguments under `:cljd/opts`, at the top level or in an alias:

```clojure
:cljd/opts {:kind :flutter
            :main my-app.main
            :dart-test-args ["--reporter" "expanded"]}
```

Two active aliases that set different `:dart-test-args` stop the run with `Two different values provided in :cljd/opts for :dart-test-args`, although upstream `doc/TESTING.md` says they are concatenated. In an alias that adds arguments to the ones already set, write `:extra-dart-test-args`, which appends. Arguments after `--` on the command line replace the merged `:dart-test-args`. Write `++` instead of `--` to append to them.

## Gotchas

- Test namespaces end in `-test` by convention, but `clojure -M:cljd test` does not discover tests by name. It compiles every `.cljd` and `.cljc` namespace in the classpath roots inside the project directory, which are `:paths` and the `:extra-paths` of the aliases on the command line, and runs the ones that define tests.
- Without the alias that adds `test/`, `clojure -M:cljd test` compiles none of the test namespaces there, but `flutter test` still runs the compiled files that an earlier run left in `test/cljd-out/`. The run can pass on stale tests. With no earlier output, it exits 1 with `Test directory "test" does not appear to contain any test files.`
- `flutter create`, which `clojure -M:cljd init` runs, writes `test/widget_test.dart` for its counter template. `clojure -M:cljd test` runs it with every other file under `test/`, and it fails with `Couldn't find constructor 'MyApp'`. Delete it.
- Widget tests need `flutter_test` in `dev_dependencies` in `pubspec.yaml`. Flutter projects include this by default. `clojure -M:cljd test` adds the `test` package to `dev_dependencies` when it is missing.
- Integration tests need `integration_test` from the Flutter SDK in `dev_dependencies`. Add it with `flutter pub add "dev:integration_test:{sdk: flutter}"`, because `flutter pub add --dev integration_test` resolves a pub.dev package and fails version solving. Integration tests run on a device, emulator, or desktop target.
- `dart test` warns `A tag was used that wasn't specified in dart_test.yaml.` (or `Tags were used that weren't specified in dart_test.yaml.` for several) for tags that `dart_test.yaml` does not declare. The tests still run.
