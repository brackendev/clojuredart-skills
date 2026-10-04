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
                      :main-opts ["-m" "cljfmt.main"]}}
 :cljd/opts {:kind :flutter
             :main my-app.main}}
```

ClojureDart releases are named `0.9.YYYYMMDD` (with an optional lowercase suffix for a second release on the same day). Each release publishes a `deps.latest.edn` asset pinned to its tag and SHA at `https://github.com/Tensegritics/ClojureDart/releases/latest/download/deps.latest.edn`. `clj -M:cljd upgrade` reads that asset and rewrites the coordinate. Pin releases rather than the `HEAD` commit.

The `.cljfmt.edn` template, including the `:file-pattern` that makes cljfmt format `.cljd` files, is in the `cljd-new` skill.

## Tooling

| Tool | Command | Purpose |
|------|---------|---------|
| ClojureDart (dev) | `clj -M:cljd flutter` | Compile, watch, hot reload |
| ClojureDart (AOT) | `clj -M:cljd compile` | Compile for deployment |
| ClojureDart (init) | `clj -M:cljd init` | Initialize project |
| ClojureDart (update) | `clj -M:cljd upgrade` | Move to the latest ClojureDart release |
| clj-kondo | `clj-kondo --lint src` | Lint `.cljd` files |
| cljfmt | `clj -M:cljfmt fix` | Format `.cljd` files |
| Babashka | `bb script.bb` | Build scripts and automation |
