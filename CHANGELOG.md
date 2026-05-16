# Changelog

## [Unreleased]

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
