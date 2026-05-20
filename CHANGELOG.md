# Changelog

## [Unreleased]

## [0.1.5] - 2026-05-20

### Changed

- The `cljd-tidy` skill is renamed to `cljd-fix` to adopt the noun-first canonical naming pattern (`<target>-<verb>`) shared across the agent-skills family. The verb suffix `-fix` consistently signals a mutating quality pipeline (lint, format, compile, dry). Operators with a saved `/cljd-tidy` invocation should replace it with `/cljd-fix`. The skill's behavior is unchanged; only the name moves.
- The `cljd-nav` skill is renamed to `clojuredart-nav` so the prefix matches the other auto-triggered skills in this package (`clojuredart`, `clojuredart-lenses`). The skill remains model-invocable only; no slash command is exposed.

## 0.1.4

### Added

- A repo-root `CONVENTIONS.md` that defines the argument grammar, scope vocabulary, and mutation defaults every user-invocable skill in this package follows. Three rules cover argument grammar (one sanctioned flag, `--report`), scope vocabulary (`(no argument)`, `all`, `<path>`), and mutation-as-default. The document lists `/cljd-new` as the standard's positional-required exemption and includes an author checklist that runs against every migrated skill.

### Changed

- The `cljd-check` skill is renamed to `cljd-tidy`. The verb now matches the default behavior: `cljfmt fix` runs by default and rewrites files in the `format` step. Operators with a saved `/cljd-check` invocation should replace it with `/cljd-tidy`. The new `--report` flag swaps the format step for `cljfmt check`, which previews diffs without writing; `lint`, `compile`, and `dry` are pure-read of source regardless.
- The `cljd-new` skill gains a `## Arguments` section that documents its positional `<project-name>` exemption and a `## Mutation` section that lists the files it writes.
- The `cljd-test` skill replaces its `## Determine Scope` section with `## Arguments` and adds a `## Mutation` section. The argument table now exposes the `all` and `<path>` rows alongside the existing `unit` and `widget` step keywords. No `--report` flag, because preview is meaningless for a test run and scaffolding only writes after operator confirmation.
- The `cljd-upgrade` skill gains a `## Arguments` section, a `## Mutation` section, and a `--report` flag. With `--report`, the skill prints the current `:sha` and the remote `HEAD` from `git ls-remote` without writing `deps.edn` or running the compile.
- The `cljd-smells-review` placeholder gains a canonical `## Arguments` section using the core scope vocabulary (`(no argument)`, `all`, `<path>`) and an explicit pure-report classification ahead of the eventual implementation.

### Changed

- The `clojuredart-lenses` skill description and `README.md` row now reflect the upstream code-lenses default-versus-opt-in split: `grug`, `Honest Code`, `Tidy First`, and `Parse Don't Validate` are the default lenses; `APOSD` and `Legacy Code` are opt-in (`+aposd`, `+legacy-code`, or direct invocation). The body still contains APOSD and Legacy Code Flutter-specific deltas so the lens can apply them when explicitly invoked.

## 0.1.2

### Removed

- `MCP Integration` section in the `clojuredart` skill's `project-workflows.md` reference. The previous text claimed `clojure-mcp` could drive the ClojureDart REPL, but `clojure-mcp` connects over nREPL and ClojureDart exposes a socket REPL only. The host-neutral baseline already documents this scoping.

### Changed

- The `clojuredart` skill's REPL section now includes a recovery step: if the banner did not appear or the port file is missing, the build process is not running or the target is web, so restart `clj -M:cljd flutter` against a native Dart target. The note distinguishes this scenario from ordinary `nc` disconnects, which do not require a restart.

## 0.1.1

### Changed

- The `clojuredart` skill now layers on top of the host-neutral [clojure](https://github.com/brackendev/clojure-skills) baseline. It opens with an applicability block naming the override boundary (Dart interop, types, `cljd.flutter` directives, async, Dart-flavored class creation, Flutter project layout) and removes the redundant Common Patterns subsections (`Conditionals`, `Destructuring`, `Loop/Recur`, `Try/Catch`) that the baseline now covers. The remaining "Imperative Dart Object Setup" content stays as a standalone section. Added explicit positive guidance for the `defrecord` factory rule and a Gotcha noting that `with-redefs` is unavailable in ClojureDart.
- The `clojuredart-lenses` skill is now a delta-only layer over [clojure-lenses](https://github.com/brackendev/clojure-skills) (in `clojure-skills`). Each of the six philosophy sections (Grug, APOSD, Tidy First, Parse Don't Validate, Honest Code, Legacy Code) keeps only the Flutter and ClojureDart-specific additions and removes the general Clojure restatements.
- Brand color changed from Clojure logo blue (`#5881D8`) to Flutter blue (`#02569B`) across all eight skills in this package, so runtime UIs can distinguish ClojureDart guidance from the `clojure` baseline at a glance.

## 0.1.0

### Added

- Initial release. ClojureDart skills extracted from the [clojure-skills](https://github.com/brackendev/clojure-skills) package as a standalone APM plugin.
- `clojuredart` (model-invoked): Core ClojureDart knowledge including Dart interop syntax, type system, `cljd.flutter` directives, class creation, async patterns, destructuring, cells, REPL workflow, and CLI reference.
- `clojuredart-lenses` (model-invoked): Translates code-lenses design philosophies (grug, APOSD, Tidy First, Parse Don't Validate, Honest Code, Legacy Code) to idiomatic ClojureDart and Flutter patterns.
- `cljd-nav` (model-invoked): Navigation patterns for ClojureDart Flutter applications, covering named routes, `go_router` with nested navigation, the `Navigator` API for dialogs and modals, and tab navigation.
- `cljd-new` (user-invoked): Scaffolds a new ClojureDart Flutter project with `deps.edn`, entry point, formatting configuration, initial compile, and clj-kondo lint setup.
- `cljd-check` (user-invoked): Runs the ClojureDart quality pipeline (`clj-kondo` lint, `cljfmt` format, ClojureDart compile, dry4clj duplicate-form scan).
- `cljd-test` (user-invoked): Scaffolds and runs ClojureDart tests with `cljd.test`, including unit, widget, and tag-filtered modes.
- `cljd-upgrade` (user-invoked): Upgrades the `tensegritics/clojuredart` dependency in `deps.edn` to the latest commit.
- `cljd-smells-review` (user-invoked, placeholder): Reserves the command name for a future ClojureDart-specific smells review. Prints a "not yet implemented" notice and exits.
