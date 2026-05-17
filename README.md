# clojuredart-skills

ClojureDart development skills packaged as an [APM](https://github.com/microsoft/apm) plugin. One install deploys the full set to every runtime APM supports: Claude Code, Codex, OpenCode, Cursor, Copilot, Gemini, and Windsurf.

Skills follow the [Agent Skills](https://agentskills.io) open standard. Three auto-trigger from conversation context (`clojuredart`, `clojuredart-lenses`, `cljd-nav`); the rest appear as slash commands.

## Companion packages

This package covers ClojureDart on Flutter. Install alongside it as needed:

| Package | Focus |
|---------|-------|
| [clojure-skills](https://github.com/brackendev/clojure-skills) | Idiomatic Clojure style, scaffolding, quality checks, and code review. |
| [biff-skills](https://github.com/brackendev/biff-skills) | [Biff](https://biffweb.com/) web framework: scaffolding, framework conventions, deployment. Layers on top of clojure-skills. |
| [clojuredart-skills](https://github.com/brackendev/clojuredart-skills) (this package) | ClojureDart / Flutter equivalents for the Clojure toolkit. |

## Install

Install [APM](https://github.com/microsoft/apm) first if you don't already have it. Then, in a project:

```bash
apm install brackendev/clojuredart-skills --target all
```

Globally for your user account:

```bash
apm install brackendev/clojuredart-skills -g --target all
```

Update later with `apm update [-g]`. Remove with `apm uninstall brackendev/clojuredart-skills [-g]`. A local filesystem path can replace the shorthand at either scope.

## Requirements

- [Flutter SDK](https://docs.flutter.dev/get-started/install) and [ClojureDart](https://github.com/Tensegritics/ClojureDart) for any skill in this package.
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
| **clojuredart** | `.cljd` files, `deps.edn` with `tensegritics/clojuredart`, `cljd-out/` directories, Flutter integration, ClojureDart REPL usage. Covers syntax, Dart interop, widget macros, project structure, compilation, and REPL-driven development. |
| **clojuredart-lenses** | Auto-triggers alongside the [code-lenses](https://github.com/brackendev/code-lenses) plugin in ClojureDart work. Translates grug, APOSD, Tidy First, Parse Don't Validate, Honest Code, and Legacy Code reviews into ClojureDart and Flutter patterns. |
| **cljd-nav** | ClojureDart navigation discussions: named routes, `go_router`, the `Navigator` API, tab navigation, and deep linking in Flutter. |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT. See [LICENSE](LICENSE).
