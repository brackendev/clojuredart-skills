---
name: cljd-upgrade
description: "Upgrade tensegritics/clojuredart in deps.edn to the latest ClojureDart release and verify the project compiles"
argument-hint: "[--report] [all]"
user-invocable: true
disable-model-invocation: true
---

# Upgrade ClojureDart

Upgrade the ClojureDart dependency in `deps.edn` to the latest published release and verify the project compiles.

ClojureDart publishes dated GitHub Releases named `0.9.YYYYMMDD`, with an optional lowercase suffix for additional releases on the same day (`0.9.20260822a`). Each release carries a `deps.latest.edn` asset pinned to the release tag and its full Git SHA. The built-in `clj -M:cljd upgrade` command reads that asset and rewrites the `:tag` and `:sha` of the top-level `tensegritics/clojuredart` coordinate. This skill relies on the built-in command for every write to the coordinate.

## Arguments

| Input         | Target                                                                       |
|---------------|------------------------------------------------------------------------------|
| (no argument) | Upgrade `tensegritics/clojuredart` in `deps.edn` to the latest release with `clj -M:cljd upgrade`, then run `clj -M:cljd compile` to verify |
| `all`         | Same as (no argument); accepted for family consistency                       |
| `--report`    | Print the current tag and SHA and the latest release tag and SHA without writing `deps.edn` or running the compile |

The target is the project's `deps.edn`, so `<path>` rows are not part of the standard scope vocabulary for this skill.

## Mutation

Mutates `deps.edn` by default. `clj -M:cljd upgrade` replaces the `:sha` of `tensegritics/clojuredart`, replaces or adds its `:tag`, and keeps the previous values next to them as `#_` comments. The skill then runs `clj -M:cljd compile`, which writes generated Dart under `lib/cljd-out/` (build artifact). On compile failure, the skill restores the full original `deps.edn`. With `--report`, the skill writes nothing and runs no compile.

## Prerequisites

Verify `deps.edn` exists in the current directory and contains a top-level `tensegritics/clojuredart` dependency. If not, stop and tell the user.

The built-in upgrader edits only the top-level `:deps` entry in `deps.edn`, and only when it uses the upstream Git URL `https://github.com/tensegritics/ClojureDart.git` and a full 40-character `:sha`. It refuses, and leaves `deps.edn` unchanged, when the project uses `:local/root`, a fork URL, or a coordinate supplied by an alias, or when the active coordinate appears more than once. The user must update such a coordinate manually. This skill does not edit it (see step 3).

## Steps

Parse `$ARGUMENTS` to determine whether `--report` is present.

### 1. Record the Current State

Read `deps.edn`. Note the current `:sha` and `:tag` of `tensegritics/clojuredart`. `:tag` is absent in projects created before ClojureDart published releases. Unless `--report` is present, save a copy of the full file so step 4 can restore it:

```bash
cp deps.edn "${TMPDIR:-/tmp}/deps.edn.cljd-upgrade"
```

### 2. Resolve the Latest Release

```bash
curl --fail --location https://github.com/Tensegritics/ClojureDart/releases/latest/download/deps.latest.edn
```

Read `:tag` and `:sha` under `[:deps tensegritics/clojuredart]` in the downloaded file. If the download fails, report the error and stop.

If `--report` is present, print the current and latest coordinates and stop without writing `deps.edn` or running the compile:

```
ClojureDart upgrade report:
  Current: <current-tag or "no tag"> (<current-sha>)
  Latest:  <latest-tag> (<latest-sha>)
```

If the current `:sha` already equals the latest SHA and the current `:tag` equals the latest tag, report "Already at the latest release" and stop.

### 3. Upgrade

Run the built-in upgrade command:

```bash
clj -M:cljd upgrade
```

If the command exits non-zero, it prints `Upgrade aborted:` followed by the reason and leaves `deps.edn` unchanged. Common reasons are the coordinates listed under Prerequisites and a refusal to downgrade from a newer tag. Report the full message and stop. Do not edit the coordinate by hand.

Read `deps.edn` again. If its `:sha` still differs from the latest SHA, run `clj -M:cljd upgrade` once more. Projects pinned to a ClojureDart commit older than the release-based upgrader reach an intermediate bridge commit on the first run and the latest release on the second run. If the coordinate still does not match the latest release after the second run, report both coordinates and stop.

### 4. Compile

```bash
clj -M:cljd compile
```

If compilation fails, restore the saved file over `deps.edn`, report the error, and stop:

```bash
cp "${TMPDIR:-/tmp}/deps.edn.cljd-upgrade" deps.edn
```

Restoring the whole file reverts both `:tag` and `:sha`.

### 5. Report

Print a summary:

```
ClojureDart upgraded:
  Previous: <old-tag or "no tag"> (<old-sha>)
  Current:  <new-tag> (<new-sha>)
  Compile:  PASS
```

## Gotchas

- Do not pin `tensegritics/clojuredart` to the `HEAD` commit from `git ls-remote`. `HEAD` can be ahead of the latest release. A coordinate without a `:tag` gets no downgrade protection, so the next `clj -M:cljd upgrade` moves such a project back to the latest release.
