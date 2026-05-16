# ClojureDart Project Workflows

## MCP Integration

This skill is designed to work with [clojure-mcp](https://github.com/bhauman/clojure-mcp), an MCP server for REPL-driven Clojure development. When clojure-mcp is available, use its tools for REPL evaluation, namespace reloading, and file operations instead of shell commands. The REPL practices in this skill apply when using clojure-mcp.

## CLI

```bash
clj -M:cljd init       # Initialize a ClojureDart project (creates Flutter structure)
clj -M:cljd flutter    # Compile, watch, and hot reload (dev mode); press RETURN to force restart
clj -M:cljd compile    # AOT compilation (for deployment)
clj -M:cljd clean      # Clean build artifacts
clj -M:cljd upgrade    # Update to latest ClojureDart
clj -M:cljd test       # Run tests
```

## Project Structure

```
my-project/
  deps.edn              # Clojure deps with ClojureDart config
  .cljfmt.edn           # cljfmt formatting rules
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
 :deps      {org.clojure/clojure {:mvn/version "1.12.0"}
             tensegritics/clojuredart
             {:git/url "https://github.com/tensegritics/ClojureDart.git"
              :sha     "<commit-sha>"}}
 :aliases   {:cljd {:main-opts ["-m" "cljd.build"]}
             :cljfmt {:extra-deps {dev.weavejester/cljfmt {:mvn/version "0.13.0"}}
                      :main-opts ["-m" "cljfmt.main"]}}
 :cljd/opts {:kind :flutter
             :main my-app.main}}
```

## Tooling

| Tool | Command | Purpose |
|------|---------|---------|
| ClojureDart (dev) | `clj -M:cljd flutter` | Compile, watch, hot reload |
| ClojureDart (AOT) | `clj -M:cljd compile` | Compile for deployment |
| ClojureDart (init) | `clj -M:cljd init` | Initialize project |
| ClojureDart (update) | `clj -M:cljd upgrade` | Update to latest ClojureDart |
| clj-kondo | `clj-kondo --lint src` | Lint `.cljd` files |
| cljfmt | `clj -M:cljfmt fix` | Format `.cljd` files |
| Babashka | `bb script.bb` | Build scripts and automation |
