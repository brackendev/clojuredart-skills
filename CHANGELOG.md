# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.1.20] - 2026-10-24

### Fixed

- `cljd-test` declares widget, integration, and tagged tests with the inline `deftest` options `:runner (ft/testWidgets [tester])` and `:tags [...]`. `deftest` ignores `^{:runner ...}` and `^{:tags ...}` metadata, so tests written from the earlier examples passed without running their bodies, and tag selection matched nothing.
- `cljd-test` puts `:dart.test/dir` in the namespace attribute map, because the compiler ignores metadata on the namespace symbol. Widget assertions use `ft/expect` with `ft/findsOneWidget` and `ft/findsNothing`, because a finder is never nil. Compiled test files are shown under `test/cljd-out/`, a test namespace can be named after `clj -M:cljd test`, and `++` appends to `:dart-test-args` while `--` replaces them.
- The `clojuredart` skill says that `instance?` works with a literal class, shows `(Foo a b nil {} -1)` as the direct record constructor, and says that the trailing dot and `new` are accepted but optional.
- The `clojuredart` skill's Extending Classes example extends `CustomPainter` instead of `StatelessWidget`, and `clojuredart-lenses` limits `deftype` to Flutter classes that `f/widget` does not replace. Both now agree that widgets come from `f/widget`.
- `clojuredart-lenses` no longer suggests blank lines between `f/widget` directives and widget expressions, which the `clojure` skill's formatting rules and ClojureDart's own code do not use.
- `/cljd-upgrade` says that the user updates a coordinate the built-in upgrader refuses, and that the skill does not edit it.
- `/cljd-fix` adds a "skipped N vendored or generated paths" line to the report when the vendored-path filter excludes any path, and list those paths under `--report`, as Rule 4 of `CONVENTIONS.md` requires.

### Changed

- The `clojuredart` skill states each rule once. Its Common Mistakes entries that repeated a Key Rule are now part of that rule, and `cljd-test`, `cljd-upgrade`, and `clojuredart-lenses` follow the same pattern.

## [0.1.19] - 2026-10-24

### Removed

- OpenCode 1 is no longer supported. The repository no longer keeps a copy of each `SKILL.md` under `.opencode/skills/` or a per-skill allow list in `opencode.jsonc`. `.opencode/skills` is now a symlink to `.apm/skills/` instead, so an OpenCode 2 session started anywhere in the repository loads the canonical skills together with their supporting files. The `.opencode/package.json` file, which pinned the OpenCode 1 plugin package, is removed as well. APM deploys the same files as before.

### Changed

- The `CONTRIBUTING.md` acceptance steps list the skills OpenCode discovers with `opencode api GET "/api/skill?location[directory]=$PWD"`, because OpenCode 2 has no `--pure` flag or `debug skill` command.

## [0.1.18] - 2026-10-24

### Fixed

- The `CONTRIBUTING.md` acceptance steps install with `apm install --target all` in the clean temporary project. On APM 0.34.0, a bare `apm install` in that project stops with "Multiple harnesses detected" because the steps create several runtime folders first. `apm update` reuses the targets that the install records in the project's `apm.yml`.

## [0.1.17] - 2026-10-17

### Changed

- In Claude Code and Codex, the agent can now run `/cljd-fix` and `/cljd-test` from a natural-language request, such as "run the ClojureDart tests", as well as from the slash command. Their descriptions now state when to use each one. `/cljd-new`, `/cljd-upgrade`, and `/cljd-smells-fix` remain user-only.

## [0.1.16] - 2026-10-04

### Added

- The `clojuredart` skill documents the `:summon` directive that ClojureDart release `0.9.20261002` recommends over `:get` for new code. The documentation covers class, form, and keyword right-hand sides, the `:as`, `:>`, and `:value>` options, and the `:of` and `:keys` shorthand. A new gotcha explains that the upstream clj-kondo hooks do not recognize `:summon` yet, so the navigation and lenses examples keep using `:get`.
- The `clojuredart` project workflow reference documents the `--offline` option that `clj -M:cljd compile` and `clj -M:cljd watch` accept to skip `pub get`.

### Changed

- `cljd-new` pins new projects to the latest ClojureDart GitHub Release, using the tag and SHA from the release's `deps.latest.edn` asset, instead of the `HEAD` commit. The template also pins Clojure `1.12.6` and cljfmt `0.16.6`.
- `cljd-upgrade` now relies on the release-based `clj -M:cljd upgrade`, which rewrites both `:tag` and `:sha`. The skill no longer writes the `HEAD` SHA by hand. It runs the upgrader a second time for projects older than the release-based upgrader, reports `Upgrade aborted` messages without editing `deps.edn`, and restores the whole `deps.edn` when the verification compile fails. `--report` now compares the project against the latest release instead of `HEAD`.

### Fixed

- The `.cljfmt.edn` that `cljd-new` scaffolds now sets a `:file-pattern` that matches `.cljd`. Previously cljfmt skipped every ClojureDart file, so the `cljd-fix` format step passed without formatting anything. The template also replaces the `:indents` map, which overrode cljfmt's default rules and misindented `let` and `when`, with `:extra-indents` entries for `f/run`, `f/widget`, and `f/build`.
- The `cljd-fix` format step now fails with a fix hint when `.cljfmt.edn` is missing or has no `:file-pattern` that matches `.cljd`, instead of reporting a pass for a run that checked no files.
- The `cljd-fix` lint step now creates `.clj-kondo/` before importing the upstream ClojureDart clj-kondo configuration. clj-kondo copies configurations only into an existing directory, so in a project without one the import did nothing and the lint reported false unresolved-symbol errors on `cljd.flutter` directives.
- The `cljd-new` examples now use the valid package name `my_app` instead of `my-app`.

## [0.1.15] - 2026-09-26

### Changed

- The `clojuredart-nav` skill description now names the navigation work that should activate it, including `.cljd` files, `go_router` types, `Navigator` push and pop, dialogs, bottom sheets, tabs, and route parameters, so the skill loads when navigation code is being written.
- The `clojuredart` skill now tells the model to require `cljd.flutter` in namespaces that build widgets instead of in every namespace, which avoids unused-namespace lint warnings in logic-only namespaces.
- The `cljd-new` note about upstream clj-kondo hook defects now describes the affected hook revisions without referring to "recent SHAs", because the scaffold always pins the latest ClojureDart commit.

### Fixed

- The `cljd-smells-fix` notice and description no longer direct users to a `TODO.md` file that is not published with the package. The notice now points to the README and CHANGELOG.
- The `cljd-fix`, `cljd-new`, `cljd-test`, `cljd-upgrade`, and `cljd-smells-fix` skills no longer refer to a `CONVENTIONS.md` file in the repository root. That file is not present in the projects where the skills run, and each skill already states its own arguments and mutation rules.

## [0.1.14] - 2026-09-16

### Changed

- The README's runtime sentence now names Grok Build after Kiro. APM 0.28.0 added `grok-build` as a canonical target that `apm install --target all` includes and that APM auto-detects from a `.grok/` directory, so the default target set now has nine runtimes. Grok Build keeps a target-native skill directory (`.grok/skills/` at project scope, `~/.grok/skills/` at user scope), like Claude Code and Kiro, rather than the shared `.agents/skills/` directory.

## [0.1.13] - 2026-07-29

### Changed

- The README's runtime sentence now scopes its list to APM's default target set rather than claiming every runtime APM supports. APM 0.26.0 also supports Antigravity, IntelliJ, and several experimental runtimes, none of which `apm install --target all` includes. The eight runtimes named are unchanged, and the sentence adds that Antigravity works when named explicitly with `--target antigravity`.

## [0.1.12] - 2026-07-29

### Removed

- The package manifest no longer declares the top-level `target: all` field. The APM manifest schema deprecates the `all` value: a parser treats the field as though it were absent and falls through to the `--target` flag or filesystem auto-detection, and the value is scheduled to become a hard parse error in a future APM release. Removing the field makes that fall-through behavior permanent. Installation behavior is unchanged, because APM already resolved targets by auto-detection rather than from this field. The separate `compilation.target` setting is not affected.

## [0.1.11] - 2026-06-23

### Changed

- The `cljd-fix` dry step is now advisory and scoped to production source, matching `clj-fix`. It scans the source paths declared by `deps.edn` `:paths` rather than a hardcoded `src test`, reads dry4clj's `--edn` output instead of matching a status string, and reports candidates without failing the run or halting the pipeline. The step reports `PASS` when no candidates are found, `REVIEW` when it lists candidates for inspection, `ERROR` when the scan cannot run or parse, or `SKIPPED`. Test directories are excluded by default because repeated test scaffolding is usually intentional.
- The `cljd-fix` dry step now relies on `unclebob/dry4clj` for `.cljd` coverage. The upstream `source-extensions` set includes `.cljd`, so the `brackendev/dry4clj` `add-cljd-extension` fork is no longer required. The project's `:dry4clj` alias must point at a revision that carries the `.cljd` extension.
- `CONVENTIONS.md` Rule 3 now states that mutating skills apply safe edits and report findings that require judgment, and that `--report` suppresses intentional source and configuration edits while verification steps such as the compile step and the clj-kondo bootstrap may still write build artifacts and tooling configuration.

## [0.1.10] - 2026-06-15

### Changed

- Add Kiro to the README's runtime list. APM 0.20.0 added Kiro as a first-class install target included in `apm install --target all`, so the README now lists it alongside Claude Code, Codex, OpenCode, Cursor, Copilot, Gemini, and Windsurf.

## [0.1.9] - 2026-05-28

### Added

- `CONVENTIONS.md` gains Rule 4: vendored and generated paths are excluded by default from mutating skills that walk the workspace. The rule sits alongside the existing three rules (renamed from "The three rules" to "The four rules"). Two filters apply together (`.gitignore` matches plus a hardcoded floor of dependency directories, build outputs, and lock files). The override rides on Rule 1's existing `<path>` `<glob>` grammar; no new flag is introduced. Applies to `/cljd-fix` and `/cljd-smells-fix`. Exempt: `/cljd-new` is scaffolding, `/cljd-upgrade` rewrites the ClojureDart `:sha` in `deps.edn`, `/cljd-test` runs the test suite against existing test files. The `lint` step's clj-kondo bootstrap write and the `compile` step's writes to `lib/cljd-out/` are tooling and build artifacts that are outside the source-mutation scope of the rule.
- `/cljd-fix` and `/cljd-smells-fix` each extend their `## Mutation` (or placeholder body) section with a paragraph restating Rule 4 in context. Operators see the policy without leaving the skill's page.

## [0.1.8] - 2026-05-26

### Fixed

- Quote the YAML `description` and `argument-hint` frontmatter in all skills that had unquoted values. Prevents potential YAML misinterpretation of special characters (angle brackets in `argument-hint`).

## [0.1.7] - 2026-05-22

### Changed

- The `cljd-smells-review` placeholder is renamed to `cljd-smells-fix` to match the mutating contract adopted by the new `clj-smells-fix` skill in [clojure-skills](https://github.com/brackendev/clojure-skills). The placeholder still prints a "not yet implemented" notice; the rename, argument grammar (`[path|all] [--report]`), and frontmatter all reflect the eventual fix-by-default behavior. Operators with a saved `/cljd-smells-review` invocation should replace it with `/cljd-smells-fix`.

## [0.1.6] - 2026-05-20

### Changed

- `CONTRIBUTING.md` is aligned to the family-wide structural template. A `CONVENTIONS.md` row is added to the Layout table, and the Skill conventions section begins with the canonical pointer paragraph.

## [0.1.5] - 2026-05-20

### Changed

- The `cljd-tidy` skill is renamed to `cljd-fix` to adopt the noun-first canonical naming pattern (`<target>-<verb>`) shared across the agent-skills family. The verb suffix `-fix` consistently signals a mutating quality pipeline (lint, format, compile, dry). Operators with a saved `/cljd-tidy` invocation should replace it with `/cljd-fix`. The skill's behavior is unchanged; only the name moves.
- The `cljd-nav` skill is renamed to `clojuredart-nav` so the prefix matches the other auto-triggered skills in this package (`clojuredart`, `clojuredart-lenses`). The skill remains model-invocable only; no slash command is exposed.

## [0.1.4] - 2026-05-19

### Added

- A repo-root `CONVENTIONS.md` that defines the argument grammar, scope vocabulary, and mutation defaults every user-invocable skill in this package follows. Three rules cover argument grammar (one sanctioned flag, `--report`), scope vocabulary (`(no argument)`, `all`, `<path>`), and mutation-as-default. The document lists `/cljd-new` as the standard's positional-required exemption and includes an author checklist that runs against every migrated skill.

### Changed

- The `cljd-check` skill is renamed to `cljd-tidy`. The verb now matches the default behavior: `cljfmt fix` runs by default and rewrites files in the `format` step. Operators with a saved `/cljd-check` invocation should replace it with `/cljd-tidy`. The new `--report` flag swaps the format step for `cljfmt check`, which previews diffs without writing; `lint`, `compile`, and `dry` are pure-read of source regardless.
- The `cljd-new` skill gains a `## Arguments` section that documents its positional `<project-name>` exemption and a `## Mutation` section that lists the files it writes.
- The `cljd-test` skill replaces its `## Determine Scope` section with `## Arguments` and adds a `## Mutation` section. The argument table now exposes the `all` and `<path>` rows alongside the existing `unit` and `widget` step keywords. No `--report` flag, because preview is meaningless for a test run and scaffolding only writes after operator confirmation.
- The `cljd-upgrade` skill gains a `## Arguments` section, a `## Mutation` section, and a `--report` flag. With `--report`, the skill prints the current `:sha` and the remote `HEAD` from `git ls-remote` without writing `deps.edn` or running the compile.
- The `cljd-smells-review` placeholder gains a canonical `## Arguments` section using the core scope vocabulary (`(no argument)`, `all`, `<path>`) and an explicit pure-report classification ahead of the eventual implementation.
- The `clojuredart-lenses` skill description and `README.md` row now reflect the upstream code-lenses default-versus-opt-in split: `grug`, `Honest Code`, `Tidy First`, and `Parse Don't Validate` are the default lenses; `APOSD` and `Legacy Code` are opt-in (`+aposd`, `+legacy-code`, or direct invocation). The body still contains APOSD and Legacy Code Flutter-specific deltas so the lens can apply them when explicitly invoked.

## [0.1.2] - 2026-05-17

### Removed

- `MCP Integration` section in the `clojuredart` skill's `project-workflows.md` reference. The previous text claimed `clojure-mcp` could drive the ClojureDart REPL, but `clojure-mcp` connects over nREPL and ClojureDart exposes a socket REPL only. The host-neutral baseline already documents this scoping.

### Changed

- The `clojuredart` skill's REPL section now includes a recovery step: if the banner did not appear or the port file is missing, the build process is not running or the target is web, so restart `clj -M:cljd flutter` against a native Dart target. The note distinguishes this scenario from ordinary `nc` disconnects, which do not require a restart.

## [0.1.1] - 2026-05-17

### Changed

- The `clojuredart` skill now layers on top of the host-neutral [clojure](https://github.com/brackendev/clojure-skills) baseline. It opens with an applicability block naming the override boundary (Dart interop, types, `cljd.flutter` directives, async, Dart-flavored class creation, Flutter project layout) and removes the redundant Common Patterns subsections (`Conditionals`, `Destructuring`, `Loop/Recur`, `Try/Catch`) that the baseline now covers. The remaining "Imperative Dart Object Setup" content stays as a standalone section. Added explicit positive guidance for the `defrecord` factory rule and a Gotcha noting that `with-redefs` is unavailable in ClojureDart.
- The `clojuredart-lenses` skill is now a delta-only layer over [clojure-lenses](https://github.com/brackendev/clojure-skills) (in `clojure-skills`). Each of the six philosophy sections (Grug, APOSD, Tidy First, Parse Don't Validate, Honest Code, Legacy Code) keeps only the Flutter and ClojureDart-specific additions and removes the general Clojure restatements.
- Brand color changed from Clojure logo blue (`#5881D8`) to Flutter blue (`#02569B`) across all eight skills in this package, so runtime UIs can distinguish ClojureDart guidance from the `clojure` baseline at a glance.

## [0.1.0] - 2026-05-17

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
