---
name: cljd-test
description: Run ClojureDart tests with cljd.test; offers to scaffold when none exist
argument-hint: "[unit|widget|all|<path>]"
user-invocable: true
disable-model-invocation: true
---

# ClojureDart Test

Run tests for ClojureDart projects using `cljd.test`. When no test files exist, the skill offers to scaffold them. See `CONVENTIONS.md` in the repo root for the argument grammar this skill follows.

## Arguments

| Input             | Target                                                                       |
|-------------------|------------------------------------------------------------------------------|
| (no argument)     | Run all tests (`clj -M:cljd test`)                                           |
| `all`             | Same as (no argument); accepted for family consistency                       |
| `<path>` `<glob>` | Restrict to those compiled Dart test files (for example, `test/my_app/core_test.dart`) |
| `unit`            | Run unit-tagged tests only (`clj -M:cljd test -- --tags unit`)               |
| `widget`          | Run widget-tagged tests only (`clj -M:cljd test -- --tags widget`)           |

The tag keywords assume tests use `:tags` metadata (see Test Tags below). If the project does not use tags, `unit` runs all tests excluding widget test files, and `widget` runs only test files that require `flutter_test`.

This skill omits `--report` because preview is meaningless for a test run; the operator can inspect the test files and tags before invoking.

## Mutation

Running tests does not write source. The test runner compiles `.cljd` test files to `.dart` under the standard Flutter test output paths; that is a build-artifact write, not source mutation. When no test files exist, the skill offers to scaffold them (see Scaffold Tests below) and writes source files only after operator confirmation.

## Run Tests

Pass additional arguments to `dart test` after `--`:

```bash
# Run a specific test file
clj -M:cljd test -- test/my_app/core_test.dart

# Run with verbose output
clj -M:cljd test -- --reporter expanded

# Exclude slow tests
clj -M:cljd test -- --exclude-tags slow
```

Report results:

```
ClojureDart Test Results:
  Status: PASS/FAIL
  Output: <test output>
```

If no test files exist, offer to scaffold them (see below).

## Scaffold Tests

When scaffolding, create test files that match the project's source structure.

### Unit Test

For a source file `src/my_app/utils.cljd`, create `src/my_app/utils_test.cljd`:

```clojure
(ns my-app.utils-test
  (:require
   [cljd.test :refer [deftest is testing are]]
   [my-app.utils :as utils]))

(deftest test-example
  (testing "describe what is being tested"
    (is (= expected (utils/my-function input)))))
```

Test files compile to the `test/` directory by default. The compiled Dart test files land in `test/` and are picked up by `dart test`.

### Widget Test

Widget tests require the `flutter_test` runner. Create `src/my_app/widgets_test.cljd`:

```clojure
(ns my-app.widgets-test
  (:require
   [cljd.test :refer [deftest is testing]]
   ["package:flutter/material.dart" :as m]
   ["package:flutter_test/flutter_test.dart" :as ft]))

(deftest ^{:runner ft/testWidgets} test-widget-renders
  (fn [^ft/WidgetTester tester]
    (await
     (.pumpWidget tester
       (m/MaterialApp
         .home (m/Scaffold
                 .body (m/Text "Hello")))))
    (is (some? (ft/find.text "Hello")))
    (is (not (some? (ft/find.text "Missing"))))))
```

Key differences from unit tests:
- Use `^{:runner ft/testWidgets}` metadata on the deftest
- Test function receives a `WidgetTester` parameter
- Use `await` with `.pumpWidget` to render widgets
- Use `ft/find.text`, `ft/find.byType`, `ft/find.byKey` to locate widgets
- Use `await (.tap tester finder)` and `await (.pumpAndSettle tester)` for interactions

### Integration Test

To compile tests into the `integration_test/` directory instead of `test/`, add namespace metadata:

```clojure
(ns ^{:dart.test/dir "integration_test"} my-app.integration-test
  (:require
   [cljd.test :refer [deftest is]]
   ["package:flutter_test/flutter_test.dart" :as ft]
   ["package:integration_test/integration_test.dart" :as it]))

(deftest ^{:runner ft/testWidgets} test-full-flow
  (fn [^ft/WidgetTester tester]
    (it/IntegrationTestWidgetsFlutterBinding.ensureInitialized)
    ;; ... full app test
    ))
```

Run integration tests with:

```bash
clj -M:cljd test -- integration_test
```

## Test Tags

Use `:tags` metadata to categorize tests:

```clojure
(deftest ^{:tags [:unit :fast]} test-pure-logic
  (is (= 4 (+ 2 2))))

(deftest ^{:tags [:widget :slow]} ^{:runner ft/testWidgets} test-complex-widget
  (fn [^ft/WidgetTester tester]
    ;; ...
    ))
```

Run tagged subsets:

```bash
clj -M:cljd test -- --tags unit
clj -M:cljd test -- --exclude-tags slow
```

## deps.edn Test Configuration

Add default test arguments in `deps.edn`:

```clojure
:cljd/opts {:kind :flutter
            :main my-app.main
            :dart-test-args ["--reporter" "expanded"]}
```

## Gotchas

- Test namespaces must end in `-test` by convention, but the compiler does not enforce this. Use it consistently for `clj -M:cljd test` to discover tests.
- Widget tests need `flutter_test` in `dev_dependencies` in `pubspec.yaml`. Flutter projects include this by default.
- Integration tests need `integration_test` in `dev_dependencies`. Add it with `flutter pub add --dev integration_test`.
- The `:runner` metadata goes on the `deftest`, not on the test function.
- `await` calls inside widget tests are required for `.pumpWidget`, `.tap`, `.pumpAndSettle`, and other async tester methods.
- `ft/find` is a top-level getter, not a class. Access finders as `ft/find.text`, `ft/find.byType`, etc.
