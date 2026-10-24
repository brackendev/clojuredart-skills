# ClojureDart Project Workflows

## CLI

```bash
clj -M:cljd init       # Initialize a ClojureDart project (creates Flutter structure)
clj -M:cljd flutter    # Compile, watch, and hot reload (dev mode); press RETURN to force restart
clj -M:cljd compile    # AOT compilation (for deployment)
clj -M:cljd compile --offline  # Compile without running `pub get` (also accepted by `watch`)
clj -M:cljd clean      # Clean build artifacts
clj -M:cljd upgrade    # Move deps.edn to the latest ClojureDart GitHub Release (tag + SHA)
clj -M:cljd test       # Run tests
```

## Project Structure

```
my-project/
  deps.edn              # Clojure deps with ClojureDart config
  .cljfmt.edn           # cljfmt formatting rules (needs a :file-pattern matching .cljd)
  src/
    my_app/
      main.cljd          # Entry point
      feature.cljd       # Feature modules
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
             :cljfmt {:extra-deps {dev.weavejester/cljfmt {:mvn/version "0.16.6"}}
                      :main-opts ["-m" "cljfmt.main"]}
             :dry4clj {:extra-deps {io.github.unclebob/dry4clj
                                    {:git/url "https://github.com/unclebob/dry4clj.git"
                                     :git/sha "5994ef2bcaaae8d12bcd0afb4f091ae09083228c"}}
                       :main-opts ["-m" "dry4clj.core"]}}
 :cljd/opts {:kind :flutter
             :main my-app.main}}
```

ClojureDart releases are named `0.9.YYYYMMDD` (with an optional lowercase suffix for a second release on the same day). Each release publishes a `deps.latest.edn` asset pinned to its tag and SHA at `https://github.com/Tensegritics/ClojureDart/releases/latest/download/deps.latest.edn`. `clj -M:cljd upgrade` reads that asset and rewrites the coordinate. Pin releases rather than the `HEAD` commit. For a new project, read the coordinate under `[:deps tensegritics/clojuredart]` in that asset, and check that `:tag` matches `0.9.YYYYMMDD` (with the optional suffix) and that `:sha` is a 40-character hexadecimal SHA before writing it.

The `:main` namespace uses hyphens (`my-app.main`), and the source path uses underscores (`src/my_app/main.cljd`). The project name must be a valid Dart package name: lowercase, with underscores and no hyphens. `dry4clj` is not published to Clojars, so the `:dry4clj` alias uses a Git coordinate pinned to a commit that scans `.cljd` files. The `cljd-fix` duplicate-form step uses that alias.

## New Project Setup

Create the directory and `deps.edn` above, then run `clj -M:cljd init` inside the directory. It creates the Flutter project structure (`pubspec.yaml`, `lib/`, `android/`, `ios/`, and the rest). Then complete the project-level configuration:

1. Add `lib/cljd-out/**` to the analyzer exclude list in `analysis_options.yaml`, keeping the existing content.
2. Append `lib/cljd-out/` and `.cpcache/` to `.gitignore` when they are not already present.
3. Create `.cljfmt.edn`:

   ```clojure
   {:paths ["src"]
    :file-pattern #re "\\.(?:clj[csxd]?|bb)$"
    :extra-indents {cljd.flutter/widget [[:inner 0]]
                    cljd.flutter/build [[:inner 0]]
                    cljd.flutter/run [[:inner 0]]}}
   ```

   cljfmt's default `:file-pattern` matches `.clj`, `.cljc`, `.cljs`, `.cljx`, and `.bb` but not `.cljd`, so without this key `cljfmt check` and `cljfmt fix` skip every ClojureDart file and still exit 0. The regular expression needs the doubled backslash because EDN strings do not accept `\.`. Use `:extra-indents`, not `:indents`, because `:indents` replaces cljfmt's default rules and misindents `let`, `when`, and other core forms.
4. Set up clj-kondo as described in the next section.
5. Run `clj -M:cljd flutter` to compile, watch, and hot reload. The first run downloads the ClojureDart dependencies and can take a minute.

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
;;   clj-kondo --copy-configs --dependencies --lint "$(clj -Spath)"
;;
;; This file layers a hook for cljd.test/deftest (upstream does not cover it),
;; adds Dart interop exclusions, and downgrades the :flutter/widget custom
;; linter so drift in cljd directive names surfaces as a warning, not an error.

{:hooks
 {:analyze-call
  {cljd.test/deftest hooks.cljd-test/deftest}}

 :linters
 {:flutter/widget {:level :warning}

  :unresolved-symbol
  {:exclude [String? DateTime? int? double?]}

  :unresolved-namespace
  {:exclude [Uri int double DateTime]}}}
```

Import the upstream exports. clj-kondo copies configurations only into an existing `.clj-kondo/` directory, which the files above create:

```bash
clj-kondo --copy-configs --dependencies --lint "$(clj -Spath)"
```

This writes `.clj-kondo/imports/tensegritics/clojuredart/`, which clj-kondo loads automatically. Commit `.clj-kondo/` to version control.

Some upstream `flutter2` hook revisions do not recognize the `:default`, `:value>`, and `:dispose-value` options to `:watch`, and some `cljd-core` hook revisions mishandle 3-form catches without a body. If clj-kondo reports `unknown keyword option to :watch` for those keywords, or false "unused binding" warnings inside a `(catch Exception e body)` with exactly three forms, patch `.clj-kondo/imports/tensegritics/clojuredart/hooks/flutter2.clj` and `cljd_core.clj`, and report the problem upstream.

## Tooling

The ClojureDart commands are listed under CLI above. The other tools:

| Tool | Command | Purpose |
|------|---------|---------|
| clj-kondo | `clj-kondo --lint src` | Lint `.cljd` files |
| cljfmt | `clj -M:cljfmt fix` | Format `.cljd` files |
| Babashka | `bb script.bb` | Build scripts and automation |
