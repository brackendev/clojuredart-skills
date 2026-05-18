---
name: cljd-new
description: Scaffold a new ClojureDart Flutter project
argument-hint: <project-name>
user-invocable: true
disable-model-invocation: true
---

# Scaffold a ClojureDart Flutter Project

Create a new Flutter project with ClojureDart configured and ready to compile. See `CONVENTIONS.md` in the repo root for the argument grammar this skill follows.

## Arguments

| Input             | Target                                                                       |
|-------------------|------------------------------------------------------------------------------|
| `<project-name>`  | Required. Use a valid Dart/Flutter package name: lowercase, underscores allowed, no hyphens (for example, `my_app`). The skill maps the package name to a Clojure namespace by replacing underscores with hyphens (`my-app.main`). |
| (no argument)     | Prompt the operator for a project name.                                      |

This skill is exempt from the `all` and `<path>` rows of the standard scope vocabulary because scaffolding has no useful default scope. See `CONVENTIONS.md` for the standard.

## Mutation

Mutates by default: creates the project directory and writes `deps.edn`, the entry-point source file `src/<project_name>/main.cljd`, `.cljfmt.edn`, `.clj-kondo/config.edn`, `.clj-kondo/hooks/cljd_test.clj`, the upstream `.clj-kondo/imports/tensegritics/clojuredart/` exports, and the standard Flutter project tree produced by `clj -M:cljd init` (`pubspec.yaml`, `lib/`, `android/`, `ios/`, and so on). Also updates `analysis_options.yaml` and appends ClojureDart entries to `.gitignore`. No `--report` flag; preview the side effects by reading this `SKILL.md`.

## Prerequisites

Verify these are installed before proceeding. If any are missing, stop and tell the user.

- `flutter` (Flutter SDK)
- `clj` (Clojure CLI / tools.deps)
- `clj-kondo` (ClojureDart linter)

Check with:

```bash
flutter --version
clj --version
clj-kondo --version
```

## Steps

### 1. Get Project Name

Use `$ARGUMENTS` as the project name. If empty, ask the user for a project name.

The project name must be a valid Dart/Flutter package name: lowercase, underscores allowed, no hyphens.

### 2. Create Project Directory and deps.edn

Create the project directory and `deps.edn` file.

Fetch the latest ClojureDart commit SHA:

```bash
git ls-remote https://github.com/tensegritics/ClojureDart.git HEAD
```

The namespace for `:main` should match the project name with hyphens replacing underscores (e.g., project `my_app` uses namespace `my-app.main`).

```bash
mkdir <project-name>
```

Create `<project-name>/deps.edn`:

```clojure
{:paths     ["src"]
 :deps      {org.clojure/clojure {:mvn/version "1.12.0"}
             tensegritics/clojuredart
             {:git/url "https://github.com/tensegritics/ClojureDart.git"
              :sha     "<latest-sha>"}}
 :aliases   {:cljd {:main-opts ["-m" "cljd.build"]}
             :cljfmt {:extra-deps {dev.weavejester/cljfmt {:mvn/version "0.13.0"}}
                      :main-opts ["-m" "cljfmt.main"]}}
 :cljd/opts {:kind :flutter
             :main <namespace>.main}}
```

### 3. Initialize the Project

Run `clj -M:cljd init` from inside the project directory. This creates the Flutter project structure (pubspec.yaml, lib/, android/, ios/, etc.):

```bash
cd <project-name>
clj -M:cljd init
```

### 4. Create Entry Point

Create `src/<project_name>/main.cljd` with a basic Flutter app:

```clojure
(ns <namespace>.main
  (:require
   ["package:flutter/material.dart" :as m]
   [cljd.flutter :as f]))

(defn main []
  (f/run
    (m/MaterialApp .title "<Project Name>")
    .home
    (m/Scaffold
      .appBar (m/AppBar .title (m/Text "<Project Name>")))
    .body
    m/Center
    (m/Text "Hello from ClojureDart!"
      .style (m/TextStyle .fontSize 24.0))))
```

Where `<namespace>` uses hyphens (Clojure convention) and `<project_name>` uses underscores (Dart convention).

### 5. Create `.cljfmt.edn`

```clojure
{:paths ["src"]
 :indents {ns [[:inner 0]]
           defn [[:inner 0]]
           fn [[:inner 0]]}}
```

### 6. Update `analysis_options.yaml`

Add `lib/cljd-out/**` to the analyzer exclude list. Read the existing file first and add the entry, preserving existing content.

### 7. Update `.gitignore`

Append these entries if not already present:

```
# ClojureDart
lib/cljd-out/
.cpcache/
```

### 8. Set Up clj-kondo

ClojureDart ships clj-kondo hooks for `cljd.flutter/widget`, `cljd.flutter/build`, `cljd.flutter/run`, and `clojure.core/try`. Import them into the project.

Create `.clj-kondo/hooks/cljd_test.clj` (upstream does not cover `cljd.test/deftest`):

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

Import upstream exports:

```bash
clj-kondo --copy-configs --dependencies --lint "$(clj -Spath)"
```

This materializes `.clj-kondo/imports/tensegritics/clojuredart/`, which clj-kondo auto-loads. Commit `.clj-kondo/` to version control.

The upstream `flutter2` hook at recent SHAs does not recognize the `:default`, `:value>`, and `:dispose-value` options to `:watch`, nor does the `cljd-core` hook handle 3-form catches without a body correctly. If clj-kondo produces `unknown keyword option to :watch` warnings for those keywords, or false "unused binding" warnings inside `(catch Exception e body)` where the catch has exactly three forms, patch `.clj-kondo/imports/tensegritics/clojuredart/hooks/flutter2.clj` and `cljd_core.clj`. Upstream fixes are the right long-term answer.

### 9. Compile and Run

```bash
clj -M:cljd flutter
```

This compiles ClojureDart, watches for changes, and hot reloads. The first run downloads ClojureDart dependencies and may take a minute.

### 10. Report

Print the created files and next steps:

```
Created:
  deps.edn
  src/<project_name>/main.cljd
  .cljfmt.edn
  .clj-kondo/config.edn
  .clj-kondo/hooks/cljd_test.clj
  .clj-kondo/imports/tensegritics/clojuredart/ (upstream exports)
  Updated analysis_options.yaml
  Updated .gitignore

Next steps:
  cd <project-name>
  clj -M:cljd flutter       # Dev mode with hot reload
  # Edit .cljd files in src/ -- changes hot reload automatically
  clj -M:cljd compile       # AOT compile for deployment
  clj -M:cljd upgrade       # Update ClojureDart version
```
