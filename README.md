# clojuredart-skills

ClojureDart development skills packaged as an [APM](https://github.com/microsoft/apm) plugin. One install deploys the full set to every runtime in APM's default target set: Claude Code, Codex, OpenCode, Cursor, Copilot, Gemini, Windsurf, Kiro, and Grok Build. Antigravity is supported by naming it explicitly with `--target antigravity`.

Skills follow the [Agent Skills](https://agentskills.io) open standard. Three auto-trigger from conversation context (`clojuredart`, `clojuredart-lenses`, `clojuredart-nav`). The rest appear as slash commands. The agent can also run `/cljd-fix` and `/cljd-test` from a natural-language request. `/cljd-upgrade` runs only as a typed slash command. Runtimes differ in how they offer these commands: Codex lists them under `/skills` and runs them as `$name`, OpenCode attaches them with `@name`, and Gemini CLI does not read the user-only setting, so its agent can start any of them.

This package layers on top of the host-neutral [clojure-skills](https://github.com/brackendev/clojure-skills) baseline. Install both together so the `clojure` skill handles general Clojure family style (naming, threading, collections, atoms, dispatch, formatting, namespaces, testing) and this package covers Dart interop, type hints and nullability, `cljd.flutter` directives, async, FFI, the ClojureDart socket REPL (which upstream labels beta), and Flutter project layout.

## Companion packages

Seven sibling APM packages. This package layers on top of [clojure-skills](https://github.com/brackendev/clojure-skills) and covers Dart interop, type hints, `cljd.flutter` directives, async, FFI, and the Flutter project workflow. Install both together for ClojureDart work. The JVM, ClojureScript, Biff, Electric, and Fulcro packages are not required for ClojureDart-only projects.

| Package | Focus | Layers on |
|---------|-------|-----------|
| [clojure-skills](https://github.com/brackendev/clojure-skills) | Host-neutral [Clojure](https://clojure.org/) family baseline: style, naming, threading, collections, atoms, dispatch, formatting, namespaces, and testing. Triggers on `.clj`, `.cljs`, `.cljc`, `.cljd`. | — |
| [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills) | [Clojure](https://clojure.org/) on the JVM: Java interop, refs / agents / STM, `with-open`, JVM-typed exceptions, `alter-var-root`, and the Clojure CLI / `tools.build` / `clj-kondo` / `cljfmt` / `test-runner` / nREPL workflow. | `clojure-skills` |
| [clojurescript-skills](https://github.com/brackendev/clojurescript-skills) | [ClojureScript](https://clojurescript.org/) style and tooling: JavaScript interop, externs inference, macro stage separation, `catch :default`, JS-flavored numbers and truthiness, and the `cljs.main` workflow. Triggers on `.cljs`, `.cljc` compiled to JS, `shadow-cljs.edn`, `figwheel-main.edn`. | `clojure-skills` |
| [biff-skills](https://github.com/brackendev/biff-skills) | [Biff](https://biffweb.com/) 2 web framework on the JVM: conventions and deployment. | `clojure-skills` + `clojure-jvm-skills` |
| [electric-skills](https://github.com/brackendev/electric-skills) | [Electric](https://github.com/hyperfiddle/electric) v3 full-stack reactive framework: `e/defn` components, `e/client` / `e/server` site transfer, `e/for` over `e/diff-by`, `e/watch` state, `e/Token` transactions, `e/Offload`, the `electric-dom3` and `electric-forms5` APIs, and the single-JVM shadow-cljs workflow. Triggers on `hyperfiddle.electric3`, `hyperfiddle.electric-dom3`, `e/defn`, `e/client`, `e/server`, and `com.hyperfiddle/electric`. | `clojure-skills` + `clojurescript-skills` + `clojure-jvm-skills` |
| [fulcro-skills](https://github.com/brackendev/fulcro-skills) | [Fulcro](https://github.com/fulcrologic/fulcro) 3 full-stack framework: `defsc` components, idents and the normalized client database, mutations, `df/load!`, dynamic routing, forms, UI state machines, Fulcro Inspect, and the Pathom 3 server. Triggers on `com.fulcrologic.fulcro.*`, `com.fulcrologic.rad.*`, `com.wsscode.pathom3.*`, `defsc`, `defmutation`, `defrouter`, `df/load!`, and ident vectors. | `clojure-skills` + `clojurescript-skills` + `clojure-jvm-skills` |
| [clojuredart-skills](https://github.com/brackendev/clojuredart-skills) (this package) | [ClojureDart](https://github.com/Tensegritics/ClojureDart) on [Flutter](https://flutter.dev/): Dart interop, type hints, `cljd.flutter` directives, async, FFI, REPL, and the Flutter project workflow. Triggers on `.cljd`, `cljd.flutter`. | `clojure-skills` |

## Install

Install [APM](https://github.com/microsoft/apm) first if you don't already have it. Then, in a project:

```bash
apm install brackendev/clojuredart-skills --target all
apm install brackendev/clojure-skills --target all
```

Globally for your user account:

```bash
apm install brackendev/clojuredart-skills -g --target all
apm install brackendev/clojure-skills -g --target all
```

Update later with `apm update [-g]`. Remove with `apm uninstall brackendev/clojuredart-skills [-g]`. A local filesystem path can replace the shorthand at either scope.

## Requirements

- The [Clojure CLI](https://clojure.org/guides/install_clojure), [ClojureDart](https://github.com/Tensegritics/ClojureDart), and the latest stable [Flutter SDK](https://docs.flutter.dev/get-started/install) for any skill in this package. The ClojureDart Flutter quick start asks for the latest stable Flutter release.
- [clojure-skills](https://github.com/brackendev/clojure-skills) installed alongside, for the host-neutral baseline.
- The `cljd-fix` format step uses [cljfmt](https://github.com/weavejester/cljfmt) through a `:cljfmt` alias in `deps.edn`, the Leiningen plugin, the `cljfmt` binary, or the cljfmt Clojure CLI tool. Without any of them, the format step reports that it was skipped and shows the alias to add.
- The `cljd-fix` dry step requires a [dry4clj](https://github.com/unclebob/dry4clj) `:dry4clj` alias in `deps.edn`. dry4clj is not published to Clojars, so the alias uses a Git coordinate. Without the alias, the dry step reports that it was skipped and shows the alias to add.

## Command guide

A quick guide to every slash command. The detailed entries under [Skills](#skills) cover arguments and examples.

| Command | Use it when | What it does |
|---------|-------------------|--------------|
| `/cljd-fix` | Lint, format, or compilation needs attention | Runs the fix pipeline, rewriting `.cljd` files in the format step with `cljfmt fix` |
| `/cljd-test` | Running ClojureDart tests | Runs `cljd.test` suites and offers to scaffold tests when none exist |
| `/cljd-upgrade` | ClojureDart is behind the latest release | Lists the release-note changes that affect the project, moves `tensegritics/clojuredart` to the latest release tag and SHA, and verifies with a compile |

## Skills

User-invocable skills share an argument grammar, scope vocabulary, and mutation default. See [CONVENTIONS.md](CONVENTIONS.md) for the full standard. In short: skills accept natural-language keywords and bare paths; the single sanctioned flag is `--report`; mutating skills apply changes by default.

### Quality and testing

#### `/cljd-fix [lint|format|compile|dry] [--report] [all]`

Fix a ClojureDart project. The skill runs all four steps by default, and the `format` step rewrites `.cljd` files in place with `cljfmt fix`. It uses the first cljfmt runner the project has: a `:cljfmt` alias in `deps.edn`, the Leiningen plugin, the `cljfmt` binary, or the cljfmt Clojure CLI tool. Pass `--report` to replace the format step with `cljfmt check`, which previews diffs without writing. The format step fails with a fix hint when the cljfmt configuration has no `:file-pattern` that matches `.cljd`, because cljfmt otherwise skips ClojureDart files. The compile step also runs `dart analyze` on the Dart files that the compile wrote, so it reports errors that would stop `flutter build`, such as a `void` callback that returns a `Future`. The advisory dry step scans `.cljd` files with the pinned dry4clj commit.

```bash
/cljd-fix
/cljd-fix compile
/cljd-fix format --report
```

#### `/cljd-test [unit|widget|all|<path>]`

Run ClojureDart tests with `cljd.test`. The skill runs the tests where the project keeps them, with `clojure -M:test:cljd test` when a `:test` alias adds `test/` to the classpath. When no test files exist, the skill offers to scaffold them under `test/` and to add the `:test` alias.

```bash
/cljd-test
/cljd-test widget
```

#### `/cljd-upgrade [--report] [all]`

Upgrade ClojureDart to the latest GitHub Release. The skill reads the release notes and commits between the current and latest release and lists the changes that affect the project. It runs the built-in `clojure -M:cljd upgrade`, which rewrites the `tensegritics/clojuredart` `:tag` and `:sha` in `deps.edn`, then verifies with a compile and a `dart analyze` check of the generated Dart. It restores the original `deps.edn` if the compile fails, and asks first when only the Dart check fails. Pass `--report` to print the current and latest release coordinates and the list of changes without writing.

```bash
/cljd-upgrade
/cljd-upgrade --report
```

### Auto-triggered

These skills activate from conversation context. They cannot be invoked directly.

| Skill | Triggers |
|-------|----------|
| **clojuredart** | `.cljd` files, `deps.edn` with `tensegritics/clojuredart`, `cljd-out/` directories, Flutter integration, ClojureDart REPL usage. Covers syntax, Dart interop, widget macros, exceptions and resource cleanup, project structure, compilation, REPL-driven development, the defrecord factory rule, and the absence of `with-redefs`. Defers to the [clojure](https://github.com/brackendev/clojure-skills) baseline for general style. The skill opens with Iron Laws (CLJD-1 through CLJD-3), rules that prevent lost work and runtime failures. The agent does not write code that breaks them, and it reports a violation it finds in existing code before changing it. |
| **clojuredart-lenses** | Auto-triggers alongside the [code-lenses](https://github.com/brackendev/code-lenses) plugin in ClojureDart work. Layers on top of `clojure-lenses` and records only the Flutter-specific deltas. Covers the four default code-lenses philosophies (grug, Honest Code, Tidy First, Parse Don't Validate) and the two opt-in philosophies (APOSD, Legacy Code) that activate when their lens is added with `+aposd` or `+legacy-code` or invoked directly. |
| **clojuredart-nav** | ClojureDart navigation discussions: named routes, `go_router`, the `Navigator` API, tab navigation, and deep linking in Flutter. |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT. See [LICENSE](LICENSE).
