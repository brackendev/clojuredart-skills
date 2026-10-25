---
name: clojuredart
description: >-
  Use when writing, editing, reviewing, or discussing ClojureDart code. Triggers:
  .cljd files, deps.edn with tensegritics/clojuredart, cljd-out/ directories,
  Flutter integration, REPL usage (socket REPL on port printed by `clj -M:cljd
  flutter`), or any mention of ClojureDart. Covers syntax, Dart interop, widget
  macros, project structure, compilation, REPL-driven development, and common
  anti-patterns.
user-invocable: false
---

# ClojureDart

ClojureDart is a Clojure dialect that compiles to Dart, enabling Clojure development within Flutter projects. Dart is more strongly typed and less dynamic than Java. This lack of dynamism is the price for excellent tree-shaking and fast startups.

Layered on top of the [clojure](https://github.com/brackendev/clojure-skills) skill, which defines host-neutral Clojure family guidance and triggers across `.clj`, `.cljs`, `.cljc`, and `.cljd`. This skill applies to ClojureDart projects and overrides the baseline only in named domains: Dart interop syntax, Dart type hints and nullability, `cljd.flutter` directives and widget composition, Dart async (`await`, `^:async`, streams, isolates), Dart-flavored class creation (`reify`, `deftype`, `defrecord` with Dart features), Dart-flavored REPL workflow, Flutter project layout and compilation, and the ClojureDart-specific test runner. The baseline still governs syntax, naming, namespace hygiene, function design, threading macros, collection idioms, atom semantics, dispatch choice, formatting, and docstrings.

## Iron Laws

These rules prevent lost work and runtime failures in ClojureDart. The `clojure` skill's Iron Laws (CLJ-1 through CLJ-12) also apply. Never write code that breaks one. When existing code breaks a law, stop and report the law ID, the location, the offending code, and the correct pattern, then ask before changing code outside the current task. A law overrides any Key Rule or reference that seems to conflict with it.

1. **CLJD-1: NEVER EDIT GENERATED DART IN `lib/cljd-out/`.** The ClojureDart book calls it generated code, and the compiler rewrites it on every compile, even without a source change. Make every change in the `.cljd` sources (`references/project-workflows.md`).
2. **CLJD-2: TREAT EVERY DYNAMIC WARNING AS A BUILD ERROR.** A build with dynamic warnings exits with status 0 but can fail at runtime, for example with `NoSuchMethodError`. The ClojureDart book: "it's best not to ship production builds with any dynamic warnings at all." Fix the first warning first, because they cascade, and add `{:no-dynamic true}` to a namespace's attribute map to make its warnings compile errors. The Gotchas section below shows how to read a warning (`references/dart-interop.md`).
3. **CLJD-3: HINT `^Type?` WHEN A VALUE CAN BE NIL.** Unlike JVM Clojure, `^String x` in ClojureDart means `x` is not allowed to be nil. Passing nil compiles without a warning and throws a type error at runtime. Write `^String? x` wherever nil is possible, and check shared `.cljc` code for JVM-style hints (`references/dart-interop.md`).

## Key Rules

1. **Use `clj -M:cljd flutter` during development.** It watches for changes and hot reloads.
2. **Use `clj -M:cljd compile` for deployment.** AOT compilation only.
3. **Exclude `lib/cljd-out/` from Dart analysis.** Add to `analysis_options.yaml` under `analyzer > exclude`.
4. **Use `^:export` for functions called from Dart.** Without it, the function is not visible to Dart code.
5. **Use `^Type` hints for Dart interop.** Prevents dynamic warnings and enables correct Dart type generation.
6. **No trailing dot on constructors.** Use `(Type args)`, not the Clojure JVM forms `(Type. args)` or `(new Type args)`. ClojureDart accepts both, but they are optional and idiomatic code omits them.
7. **Use `(->Foo a b)` / `(map->Foo {...})` factories for records.** The baseline rule applies, and calling the record constructor directly, `(Foo a b nil {} -1)`, requires three extra arguments (meta, extmap, hash) that the factory functions supply automatically. Always call the factory.
8. **`with-redefs` and `with-redefs-fn` are unavailable in ClojureDart.** The JVM testing idiom of redefining vars at test time does not translate. For test seams, pass dependencies as function arguments or expose them through `:bind` / `:get` so tests can supply alternatives.

For CLI commands, project structure, deps.edn configuration, new project setup, clj-kondo setup, and tooling, read `references/project-workflows.md`.

## References

The Iron Laws and Key Rules are the short form. Each reference file below holds the detailed guidance and the examples for one topic. Before writing, reviewing, or refactoring ClojureDart code, read every reference whose topic the task touches, and read the next one when the work moves to a new topic.

| When the task involves | Read |
|------------------------|------|
| Dart interop: properties, methods, constructors, static and extension members, type hints, nullability, parametrized types, dynamic warnings, `^:export`, destructuring Dart objects, named and optional parameters, consts, or creating classes with `reify`, `deftype`, and `defrecord` | `references/dart-interop.md` |
| `await`, async functions, streams, or isolates | `references/async.md` |
| Flutter widgets with `cljd.flutter`: `f/run`, `f/build`, `f/widget`, directives, cells, and watches | `references/flutter.md` |
| Evaluating code in the ClojureDart REPL or driving a running app from it | `references/repl.md` |
| Namespace conventions, entry points, adding Dart packages, platform channels and FFI, or conditional reading | `references/project-structure.md` |
| CLI commands, project layout, `deps.edn`, new project setup, clj-kondo setup, or tooling | `references/project-workflows.md` |

## Gotchas

### Dynamic Warnings

ClojureDart prints every dynamic warning in one format, `DYNAMIC WARNING: can't resolve member <name> on target type <type> of library <library>`, followed by the source location. The warning exits with status 0. The target type tells the two kinds apart, and they need different fixes.

**Inference failures: the target type is `dynamic`.** The compiler does not know the type, so it cannot check the call. Fix with type hints:

1. Start with the first warning (see Dynamic Warnings under Types in `references/dart-interop.md`).
2. Add `^Type` hint to the variable or parameter causing the warning.
3. Recompile and check if downstream warnings resolve.
4. Common sources: function parameters without type hints, return values from Dart APIs, destructured values from untyped maps.

```clojure
;; Warning: can't resolve member length on target type dynamic
(defn count-chars [s] (.-length s))

;; Fixed: type hint resolves it
(defn count-chars [^String s] (.-length s))
```

**Resolution failures: the target type is a real type.** The named member does not exist on that type. No type hint will fix this. Either the member name is wrong or the target's type is wrong. A common cause is a Python-style name such as `.__setitem`, `.__getitem`, or `.__contains__`, which Dart does not have. The call compiles but throws `NoSuchMethodError` at runtime. Treat these as build failures and fix at the source. To mutate a Dart Map, use `(. m "[]=" k v)` (see Operator Overloading in `references/dart-interop.md`), or prefer a Clojure atom holding a persistent map.

### Flutter Web

- `HashMap` keyword access can return `null` in release builds on Flutter Web. If you encounter nil values that work in debug mode, check whether the data flows through Dart HashMap boundaries.
- Type cast errors in release builds that do not appear in debug mode are typically caused by Dart's tree-shaking removing type information. Add explicit type hints at the boundary where data crosses from Dart to ClojureDart.
- Web builds use `clj -M:cljd compile` followed by `flutter build web`. Test in release mode (`flutter run -d chrome --release`) before deploying.

### Common Mistakes

- **`instance?` with a class that is not a literal.** Dart needs the type in an `is` check written literally. `(instance? String x)` works because it expands to `(dart/is? x String)`, but a class held in a local or var fails to compile, and `instance?` used as a function value, as in `(partial instance? String)`, throws `UnsupportedError` at runtime. Pass the class literally, or write `(dart/is? x Type)`.
- **clj-kondo noise on `cljd.flutter/widget` or `cljd.test/deftest`.** Without the project's clj-kondo hooks and upstream exports, the linter produces hundreds of false positives on widget directives and test runners. Follow the clj-kondo setup in `references/project-workflows.md`: add the `cljd.test/deftest` hook and `.clj-kondo/config.edn`, then run `clj-kondo --copy-configs --dependencies --lint "$(clj -Spath)"`.
- **clj-kondo does not understand `:summon` yet.** The upstream hooks (as of ClojureDart `0.9.20261002`) do not handle the `:summon` directive, so every name it binds is reported as an unresolved symbol, even though the code compiles. Until upstream adds hook support, either keep `:get` in linted code or accept the findings for `:summon` forms.
