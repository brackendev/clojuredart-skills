# ClojureDart Project Workflows

## CLI

```bash
clojure -M:cljd init       # Initialize a ClojureDart project (creates Flutter structure)
clojure -M:cljd flutter    # Compile, watch, and hot reload (dev mode); press RETURN to force restart
clojure -M:cljd compile    # AOT compilation (for deployment)
clojure -M:cljd compile --offline  # Compile without running `pub get` (also accepted by `watch`)
clojure -M:cljd clean      # Clean build artifacts
clojure -M:cljd upgrade    # Move deps.edn to the latest ClojureDart GitHub Release (tag + SHA)
clojure -M:test:cljd test  # Run tests (the :test alias adds test/ to the classpath)
```

Upstream ClojureDart documentation writes these commands with `clj`, as in `clj -M:cljd flutter`. `clojure` runs the same commands. The `clj` script only wraps `clojure` in `rlwrap` for line editing, and without `rlwrap` it prints `Please install rlwrap for command editing or use "clojure" instead.` and exits 1. `clojure -M:cljd flutter` still reads RETURN from standard input and restarts the app.

## Project Structure

```
my-project/
  deps.edn              # Clojure deps with ClojureDart config
  .cljfmt.edn           # cljfmt formatting rules (needs a :file-pattern matching .cljd)
  src/
    my_app/
      main.cljd          # Entry point
      feature.cljd       # Feature modules
  test/
    my_app/
      feature_test.cljd  # cljd.test namespace (on the classpath through the :test alias)
    cljd-out/            # Compiled test namespaces (generated)
  lib/
    cljd-out/            # Generated Dart (NEVER edit)
    main.dart            # Dart entry point
  pubspec.yaml           # Flutter/Dart dependencies
  analysis_options.yaml  # Dart analysis config
```

## deps.edn Configuration

```clojure
{:paths     ["src"]
 :deps      {org.clojure/clojure {:mvn/version "1.12.6"}
             tensegritics/clojuredart
             {:git/url "https://github.com/tensegritics/ClojureDart.git"
              :tag     "<release-tag>"
              :sha     "<release-sha>"}}
 :aliases   {:cljd {:main-opts ["-m" "cljd.build"]}
             :test {:extra-paths ["test"]}
             :cljfmt {:replace-deps {dev.weavejester/cljfmt {:mvn/version "0.16.6"}}
                      :main-opts ["-m" "cljfmt.main"]}
             :dry4clj {:extra-deps {io.github.unclebob/dry4clj
                                    {:git/url "https://github.com/unclebob/dry4clj.git"
                                     :git/sha "5994ef2bcaaae8d12bcd0afb4f091ae09083228c"}}
                       :main-opts ["-m" "dry4clj.core"]}}
 :cljd/opts {:kind :flutter
             :main my-app.main}}
```

ClojureDart releases are named `0.9.YYYYMMDD` after the UTC date, and later releases on the same UTC day add a suffix from `a` through `z` (`0.9.20260822a`). Each release publishes a `deps.latest.edn` asset pinned to its tag and SHA at `https://github.com/Tensegritics/ClojureDart/releases/latest/download/deps.latest.edn`. `clojure -M:cljd upgrade` reads that asset and rewrites the coordinate. Pin releases rather than the `HEAD` commit. For a new project, read the coordinate under `[:deps tensegritics/clojuredart]` in that asset, and check that `:tag` matches `0.9.YYYYMMDD` (with the optional suffix) and that `:sha` is a 40-character hexadecimal SHA before writing it.

The `:main` namespace uses hyphens (`my-app.main`), and the source path uses underscores (`src/my_app/main.cljd`). `clojure -M:cljd init` derives the Dart package name from the directory name and adds a `cljd_` prefix (directory `my_app` becomes package `cljd_my_app`), so the directory name needs no Dart-specific form. `clojure -M:cljd test` compiles only the namespaces in the project's classpath roots, so test namespaces under `test/` need the `:test` alias, and the test command is `clojure -M:test:cljd test`. The `:cljfmt` alias uses `:replace-deps` because cljfmt reads source as text and needs none of the project's dependencies. `dry4clj` is not published to Clojars, so the `:dry4clj` alias uses a Git coordinate pinned to a commit that scans `.cljd` files. The `cljd-fix` duplicate-form step uses that alias.

## New Project Setup

Create the directory and `deps.edn` above, then run `clojure -M:cljd init` inside the directory. It runs `flutter create` for the Flutter project structure (`pubspec.yaml`, `lib/`, `android/`, `ios/`, `test/`, and the rest), writes `lib/main.dart` as an export of the `:main` namespace's `main`, and appends `.cpcache/`, `.clojuredart/`, `lib/cljd-out/`, and `test/cljd-out/` to `.gitignore`. Integration test namespaces compile to `integration_test/cljd-out/`, which `init` does not add to `.gitignore` and `clojure -M:cljd clean` does not delete. Then complete the project-level configuration:

1. Add `lib/cljd-out/**` to the analyzer exclude list in `analysis_options.yaml`, keeping the existing content.
2. Delete `test/widget_test.dart`. `flutter create` writes it for its counter template, and it fails to compile (`Couldn't find constructor 'MyApp'`) on every `clojure -M:test:cljd test` run, because that command runs `flutter test` over the whole `test/` directory. Keep the `test/` directory for the `.cljd` test namespaces.
3. Create `.cljfmt.edn`:

   ```clojure
   {:paths ["src" "test"]
    :file-pattern #re "\\.(?:clj[csxd]?|bb)$"
    :extra-indents {cljd.flutter/widget [[:inner 0]]
                    cljd.flutter/build [[:inner 0]]
                    cljd.flutter/run [[:inner 0]]}}
   ```

   cljfmt's default `:file-pattern` matches `.clj`, `.cljc`, `.cljs`, `.cljx`, and `.bb` but not `.cljd`, so without this key `cljfmt check` and `cljfmt fix` skip every ClojureDart file and still exit 0. The regular expression needs the doubled backslash because EDN strings do not accept `\.`. Use `:extra-indents`, not `:indents`, because `:indents` replaces cljfmt's default rules and misindents `let`, `when`, and other core forms.
4. Set up clj-kondo as described in the next section, and add `.clj-kondo/.cache/` to `.gitignore`. When the project will have integration tests, add `integration_test/cljd-out/` as well.
5. Run `clojure -M:cljd flutter` to compile, watch, and hot reload. The first run downloads the ClojureDart dependencies and can take a minute.

## clj-kondo Setup

ClojureDart ships clj-kondo hooks for `cljd.flutter/widget`, `cljd.flutter/build`, `cljd.flutter/run`, and `clojure.core/try`. Upstream does not cover `cljd.test/deftest`, so the project adds its own hook. Without these, clj-kondo reports hundreds of false positives on widget directives and test runners.

Create `.clj-kondo/hooks/cljd_test.clj`:

```clojure
(ns hooks.cljd-test
  (:require [clj-kondo.hooks-api :as api]))

(defn- runner-bindings [runner-node]
  (when (api/list-node? runner-node)
    (some #(when (api/vector-node? %) (:children %))
          (:children runner-node))))

(defn- split [rest-forms]
  (loop [params []
         body []
         forms rest-forms]
    (cond
      (empty? forms)
      {:params params :body body}

      (and (api/keyword-node? (first forms)) (seq (rest forms)))
      (let [k (api/sexpr (first forms))
            v (second forms)]
        (if (= k :runner)
          (recur (into params (or (runner-bindings v) [])) body (drop 2 forms))
          (recur params body (drop 2 forms))))

      :else
      (recur params (conj body (first forms)) (rest forms)))))

(defn deftest [{:keys [node]}]
  (let [[_ test-name & rest-forms] (:children node)
        {:keys [params body]} (split rest-forms)
        new-node (api/list-node
                  (list*
                   (api/token-node 'clojure.core/defn)
                   test-name
                   (api/vector-node (vec params))
                   body))]
    {:node (with-meta new-node (meta node))}))
```

Create `.clj-kondo/config.edn`:

```clojure
;; Upstream cljd clj-kondo config is imported under
;; .clj-kondo/imports/tensegritics/clojuredart/ by clj-kondo --copy-configs.
;; Regenerate with:
;;   clj-kondo --copy-configs --dependencies --lint "$(clojure -Spath)"
;;
;; This file layers a hook for cljd.test/deftest (upstream does not cover it),
;; lints the other cljd.test macros as their clojure.test counterparts,
;; adds Dart interop exclusions, and downgrades the :flutter/widget custom
;; linter so drift in cljd directive names surfaces as a warning, not an error.

{:hooks
 {:analyze-call
  {cljd.test/deftest hooks.cljd-test/deftest}}

 :lint-as
 {cljd.test/is clojure.test/is
  cljd.test/are clojure.test/are
  cljd.test/testing clojure.test/testing}

 :linters
 {:flutter/widget {:level :warning}

  :unresolved-symbol
  {:exclude [String? DateTime? int? double?
             (cljd.test/is [thrown? thrown-with-msg?])]}

  :unresolved-namespace
  {:exclude [DateTime double Future int List Stream Uri]}}}
```

Without the `:lint-as` entries, clj-kondo reports the parameters of an `are` template as unresolved symbols. The `:unresolved-namespace` list covers the `dart:core` classes whose static members the skill's examples call with the slash form, such as `(Future/delayed ...)` and `(List/empty ...)`. clj-kondo merges these lists with the imported upstream configuration.

Import the upstream exports. clj-kondo copies configurations only into an existing `.clj-kondo/` directory, which the files above create:

```bash
clj-kondo --copy-configs --dependencies --lint "$(clojure -Spath)"
```

This writes `.clj-kondo/imports/tensegritics/clojuredart/`, which clj-kondo loads automatically. Commit `.clj-kondo/` to version control except `.clj-kondo/.cache/`, as the clj-kondo README recommends.

The upstream hooks in release `0.9.20261002` produce these false positives:

- `unknown keyword option to :watch` for the `:default`, `:value>`, and `:dispose-value` options. The `:flutter/widget` level above keeps them warnings.
- `Unresolved symbol` for every name that `:summon` binds (see Gotchas in `SKILL.md`).
- `unused binding` for a `catch` whose only body form is a bare symbol, as in `(catch Exception e fallback)`, because the `try` hook reads that symbol as a stack trace binding.

clj-kondo also misreads function definitions that use Dart parameter syntax (`[a b .d .e]`, `[... a 42 b]`). It reports the parameters as unused or unresolved, and calls that omit optional arguments as wrong arity. Put `#_:clj-kondo/ignore` before such a definition and before each such call instead of rewriting them.

Do not patch the files under `.clj-kondo/imports/`. Running `clj-kondo --copy-configs` again overwrites them.

## Tooling

The ClojureDart commands are listed under CLI above. The other tools:

| Tool | Command | Purpose |
|------|---------|---------|
| clj-kondo | `clj-kondo --lint src test` | Lint `.cljd` files |
| cljfmt | `clojure -M:cljfmt fix` | Format `.cljd` files |
| Dart analyzer | `dart analyze --no-fatal-warnings <generated files...>` | Check generated Dart that `flutter build` would reject (the `cljd-fix` compile step lists the files) |
| Babashka | `bb script.bb` | Build scripts and automation |
