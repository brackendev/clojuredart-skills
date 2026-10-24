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
| (no argument)     | Run all tests (`clj -M:cljd test`)                                           |
| `all`             | Same as (no argument); accepted for family consistency                       |
| `<path>` `<glob>` | Restrict to those compiled Dart test files (for example, `test/cljd-out/my-app/core-test_test.dart`) |
| `unit`            | Run unit-tagged tests only (`clj -M:cljd test -- --tags unit`)               |
| `widget`          | Run widget-tagged tests only (`clj -M:cljd test -- --tags widget`)           |

The tag keywords assume tests declare the `:tags` option (see Test Tags below). If the project does not use tags, `unit` runs all tests excluding widget test files, and `widget` runs only test files that require `flutter_test`.

This skill omits `--report` because preview is meaningless for a test run; the operator can inspect the test files and tags before invoking.

## Mutation

Running tests does not write source. The test runner compiles `.cljd` test files to `.dart` under the standard Flutter test output paths; that is a build-artifact write, not source mutation. When no test files exist, the skill offers to scaffold them (see Scaffold Tests below) and writes source files only after operator confirmation.

## Run Tests

Name namespaces after `test` to narrow the run, and pass additional arguments to `dart test` after `--`:

```bash
# Run one test namespace
clj -M:cljd test my-app.core-test

# Run a specific compiled test file
clj -M:cljd test -- test/cljd-out/my-app/core-test_test.dart

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

Test namespaces compile to `test/cljd-out/` by default (for example, `test/cljd-out/my-app/utils-test_test.dart`), where `dart test` picks them up.

### Widget Test

Widget tests require the `flutter_test` runner. Create `src/my_app/widgets_test.cljd`:

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
- Declare the runner with the inline option `:runner (ft/testWidgets [tester])` after the test name. `deftest` reads `:runner` and `:tags` only as inline options, so `^{:runner ...}` metadata is ignored and the test passes without running its body.
- The runner binds a `WidgetTester` to the name in the binding vector. Type-hint it as `^ft/WidgetTester` in the body.
- `await` every async tester method, for example `(await (.pumpWidget tester widget))`, `(await (.tap tester finder))`, and `(await (.pumpAndSettle tester))`. The body must await at least one of them: `testWidgets` expects a callback that returns a `Future`, and a body with no `await` fails with `type '...' is not a subtype of type 'Future<void>'`.
- Locate widgets with `ft/find.text`, `ft/find.byType`, and `ft/find.byKey`. `ft/find` is a top-level constant (a `CommonFinders` instance), not a class.
- Assert on finders with `(ft/expect finder ft/findsOneWidget)` or `ft/findsNothing`. A finder is never nil, so `(is (some? finder))` always passes.

### Integration Test

To compile tests into the `integration_test/` directory instead of `test/`, put `:dart.test/dir` in the namespace attribute map after the name. Metadata on the namespace symbol is ignored:

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

Run integration tests with:

```bash
clj -M:cljd test -- integration_test
```

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

Arguments after `--` on the command line replace `:dart-test-args`. Write `++` instead of `--` to append to them.

## Gotchas

- Test namespaces end in `-test` by convention, but `clj -M:cljd test` does not discover tests by name. It compiles every `.cljd` and `.cljc` namespace under the project's source paths and runs the ones that define tests.
- Widget tests need `flutter_test` in `dev_dependencies` in `pubspec.yaml`. Flutter projects include this by default.
- Integration tests need `integration_test` in `dev_dependencies`. Add it with `flutter pub add --dev integration_test`.
