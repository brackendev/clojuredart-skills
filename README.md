# clojuredart-skills

ClojureDart development skills packaged as an [APM](https://github.com/microsoft/apm) plugin. One install deploys the full set to every runtime APM supports: Claude Code, Codex, OpenCode, Cursor, Copilot, Gemini, and Windsurf.

Skills follow the [Agent Skills](https://agentskills.io) open standard. Three auto-trigger from conversation context (`clojuredart`, `clojuredart-lenses`, `cljd-nav`); the rest appear as slash commands.

This package layers on top of the host-neutral [clojure-skills](https://github.com/brackendev/clojure-skills) baseline. Install both together so the `clojure` skill handles general Clojure family style (naming, threading, collections, atoms, dispatch, formatting, namespaces, testing) and this package covers Dart interop, type hints and nullability, `cljd.flutter` directives, async, FFI, the ClojureDart socket REPL, and Flutter project layout.

## Companion packages

Six sibling APM packages. This package layers on top of [clojure-skills](https://github.com/brackendev/clojure-skills) and covers Dart interop, type hints, `cljd.flutter` directives, async, FFI, and the Flutter project workflow. Install both together for ClojureDart work. The JVM, ClojureScript, Biff, and Fulcro packages are not required for ClojureDart-only projects.

| Package | Focus | Layers on |
|---------|-------|-----------|
| [clojure-skills](https://github.com/brackendev/clojure-skills) | Host-neutral Clojure family baseline (style, naming, threading, collections, atoms, dispatch, formatting, namespaces, testing). Triggers on `.clj`, `.cljs`, `.cljc`, `.cljd`. | — |
| [clojure-jvm-skills](https://github.com/brackendev/clojure-jvm-skills) | JVM-specific Clojure (Java interop, refs / agents / STM, `with-open`, JVM-typed exceptions, `alter-var-root`, Clojure CLI / `tools.build` / `clj-kondo` / `cljfmt` / `test-runner` / nREPL workflow). | `clojure-skills` |
| [clojurescript-skills](https://github.com/brackendev/clojurescript-skills) | ClojureScript-specific style (JavaScript interop, externs inference, macro stage separation, `catch :default`, JS-flavored numbers and truthiness, the `cljs.main` workflow). Triggers on `.cljs`, `.cljc` compiled to JS, `shadow-cljs.edn`, `figwheel-main.edn`. | `clojure-skills` |
| [biff-skills](https://github.com/brackendev/biff-skills) | [Biff](https://biffweb.com/) web framework on the JVM: scaffolding, conventions, deployment. | `clojure-skills` + `clojure-jvm-skills` |
| [fulcro-skills](https://github.com/brackendev/fulcro-skills) | [Fulcro](https://github.com/fulcrologic/fulcro) full-stack framework: `defsc` components, idents and the normalized client database, mutations, `df/load!`, dynamic routing, forms, UI state machines, Fulcro Inspect, and the Pathom 3 server. Triggers on `com.fulcrologic.fulcro.*`, `com.fulcrologic.rad.*`, `com.wsscode.pathom3.*`, `defsc`, `defmutation`, `defrouter`, `df/load!`, and ident vectors. | `clojure-skills` + `clojurescript-skills` + `clojure-jvm-skills` |
| [clojuredart-skills](https://github.com/brackendev/clojuredart-skills) (this package) | ClojureDart on Flutter: Dart interop, type hints, `cljd.flutter` directives, async, FFI, REPL, Flutter project workflow. Triggers on `.cljd`, `cljd.flutter`. | `clojure-skills` |

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

- [Flutter SDK](https://docs.flutter.dev/get-started/install) and [ClojureDart](https://github.com/Tensegritics/ClojureDart) for any skill in this package.
- [clojure-skills](https://github.com/brackendev/clojure-skills) installed alongside, for the host-neutral baseline.
- The `cljd-check` dry step requires a [dry4clj](https://github.com/unclebob/dry4clj) `:dry4clj` alias in `deps.edn`.

## Skills

### Scaffolding and quality

#### `/cljd-new <project-name>`

Scaffold a new ClojureDart Flutter project, including clj-kondo lint setup.

```bash
/cljd-new my-app
```

#### `/cljd-check [lint|format|compile|dry]`

Run the ClojureDart quality pipeline. Defaults to lint, format, compile, dry. The dry step scans `.clj`, `.cljc`, and `.cljs` files; `.cljd` coverage requires the upstream dry4clj extension (see TODO).

```bash
/cljd-check
/cljd-check compile
```

#### `/cljd-test [unit|widget|all]`

Scaffold and run ClojureDart tests with `cljd.test`.

```bash
/cljd-test
/cljd-test widget
```

#### `/cljd-upgrade`

Upgrade ClojureDart to the latest version. Updates the `tensegritics/clojuredart` dependency SHA in `deps.edn`.

```bash
/cljd-upgrade
```

#### `/cljd-smells-review [scope or options...]` (placeholder)

Reserves the command name for a future ClojureDart-specific smells review. Currently prints a "not yet implemented" notice and exits. See [TODO.md](TODO.md).

### Auto-triggered

These skills activate from conversation context. They cannot be invoked directly.

| Skill | Triggers |
|-------|----------|
| **clojuredart** | `.cljd` files, `deps.edn` with `tensegritics/clojuredart`, `cljd-out/` directories, Flutter integration, ClojureDart REPL usage. Covers syntax, Dart interop, widget macros, project structure, compilation, REPL-driven development, the defrecord factory rule, and the absence of `with-redefs`. Defers to the [clojure](https://github.com/brackendev/clojure-skills) baseline for general style. |
| **clojuredart-lenses** | Auto-triggers alongside the [code-lenses](https://github.com/brackendev/code-lenses) plugin in ClojureDart work. Layers on top of `clojure-lenses` and records only the Flutter-specific deltas. Covers the four default code-lenses philosophies (grug, Honest Code, Tidy First, Parse Don't Validate) and the two opt-in philosophies (APOSD, Legacy Code) that activate when their lens is added with `+aposd` or `+legacy-code` or invoked directly. |
| **cljd-nav** | ClojureDart navigation discussions: named routes, `go_router`, the `Navigator` API, tab navigation, and deep linking in Flutter. |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT. See [LICENSE](LICENSE).
