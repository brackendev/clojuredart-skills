---
name: cljd-upgrade
description: "Upgrade tensegritics/clojuredart in deps.edn to the latest ClojureDart release, list the release-note changes that affect the project, and verify the project compiles"
argument-hint: "[--report] [all]"
user-invocable: true
disable-model-invocation: true
---

# Upgrade ClojureDart

Upgrade the ClojureDart dependency in `deps.edn` to the latest published release, read the release notes between the current and latest release, and verify the project compiles.

ClojureDart publishes dated GitHub Releases named `0.9.YYYYMMDD`, with an optional lowercase suffix for additional releases on the same day (`0.9.20260822a`). Each release carries a `deps.latest.edn` asset pinned to the release tag and its full Git SHA. The built-in `clojure -M:cljd upgrade` command reads that asset and rewrites the `:tag` and `:sha` of the top-level `tensegritics/clojuredart` coordinate. This skill relies on the built-in command for every write to the coordinate.

## Arguments

| Input         | Target                                                                       |
|---------------|------------------------------------------------------------------------------|
| (no argument) | Upgrade `tensegritics/clojuredart` in `deps.edn` to the latest release with `clojure -M:cljd upgrade`, then run `clojure -M:cljd compile` and `dart analyze` on the generated Dart to verify |
| `all`         | Same as (no argument); accepted for family consistency                       |
| `--report`    | Print the current tag and SHA, the latest release tag and SHA, and the changes listed in the release notes between them, without writing `deps.edn` or running the compile |

The target is the project's `deps.edn`, so `<path>` rows are not part of the standard scope vocabulary for this skill.

## Mutation

Mutates `deps.edn` by default. `clojure -M:cljd upgrade` replaces the `:sha` of `tensegritics/clojuredart`, replaces or adds its `:tag`, and keeps the previous values next to them as `#_` comments. The skill then runs `clojure -M:cljd compile`, which writes generated Dart under `lib/cljd-out/` (build artifact), and checks that Dart with `dart analyze`, which writes nothing to the project. On compile failure, the skill restores the full original `deps.edn`. When only the Dart check fails, it asks before restoring. With `--report`, the skill writes nothing and runs no compile.

## Prerequisites

Verify `deps.edn` exists in the current directory and contains a top-level `tensegritics/clojuredart` dependency. If not, stop and tell the user.

The built-in upgrader edits only the top-level `:deps` entry in `deps.edn`, and only when it uses the upstream Git URL `https://github.com/tensegritics/ClojureDart.git` and a full 40-character `:sha`. It refuses, and leaves `deps.edn` unchanged, when the project uses `:local/root`, a fork URL, or a coordinate supplied by an alias, or when the active coordinate appears more than once. The user must update such a coordinate manually. This skill does not edit it (see step 4).

## Steps

Parse `$ARGUMENTS` to determine whether `--report` is present.

### 1. Record the Current State

Read `deps.edn`. Note the current `:sha` and `:tag` of `tensegritics/clojuredart`. `:tag` can be absent, and the upgrader adds it. Unless `--report` is present, save a copy of the full file so step 5 can restore it:

```bash
cp deps.edn "${TMPDIR:-/tmp}/deps.edn.cljd-upgrade"
```

### 2. Resolve the Latest Release

```bash
curl --fail --location https://github.com/Tensegritics/ClojureDart/releases/latest/download/deps.latest.edn
```

Read `:tag` and `:sha` under `[:deps tensegritics/clojuredart]` in the downloaded file. If the download fails, report the error and stop.

If the current `:sha` already equals the latest SHA and the current `:tag` equals the latest tag, report "Already at the latest release" and stop.

### 3. Read the Release Notes

ClojureDart publishes its changes as GitHub release notes. List the releases:

```bash
gh release list -R Tensegritics/ClojureDart --limit 100 --json tagName,publishedAt --jq '.[] | "\(.tagName) \(.publishedAt)"'
```

For each tag after the current tag, up to and including the latest tag, read the notes:

```bash
gh release view <tag> -R Tensegritics/ClojureDart --json body --jq .body
```

GitHub generates these notes, and they often hold only a "Full Changelog" compare link. Read the commits in the whole range too, with the current `:tag` as the base, or the current `:sha` when `deps.edn` has no `:tag`:

```bash
gh api --paginate "repos/Tensegritics/ClojureDart/compare/<current-tag-or-sha>...<latest-tag>?per_page=100" --jq '.commits[] | "\(.sha[0:7]) \(.commit.message | split("\n")[0])"'
```

Without `gh`, fetch the same JSON with `curl --fail --location "https://api.github.com/repos/Tensegritics/ClojureDart/compare/<current-tag-or-sha>...<latest-tag>?per_page=100"`. If neither source can be read, say so in the report and continue.

From the notes and commit messages, list the changes that can affect the project: renamed or removed namespaces, functions, `cljd.flutter` directives, or options; changed defaults; new required configuration; and changes to the `clojure -M:cljd` commands. Search the project's `.cljd` and `.cljc` sources and `deps.edn` for each affected name, and note where it occurs. When a change requires a project edit, describe the edit and ask before making it. This skill writes only `deps.edn` on its own.

If `--report` is present, print the coordinates and the list, then stop without writing `deps.edn` or running the compile:

```
ClojureDart upgrade report:
  Current: <current-tag or "no tag"> (<current-sha>)
  Latest:  <latest-tag> (<latest-sha>)
  Changes: <each relevant change, with the files that use it, or "none found">
```

### 4. Upgrade

Run the built-in upgrade command:

```bash
clojure -M:cljd upgrade
```

If the command exits non-zero, it prints `Upgrade aborted:` followed by the reason and leaves `deps.edn` unchanged. Common reasons are the coordinates listed under Prerequisites and a refusal to downgrade from a newer tag. Report the full message and stop. Do not edit the coordinate by hand.

Read `deps.edn` again. If its `:sha` still differs from the latest SHA, run `clojure -M:cljd upgrade` once more. The command runs the project's current ClojureDart, and upstream `doc/RELEASING.md` documents that the first run can stop at an intermediate commit that the second run moves to the latest release. If the coordinate still does not match the latest release after the second run, report both coordinates and stop.

### 5. Compile

Create a marker file, and note the directory it prints. The Dart check below uses it to find the files this compile writes:

```bash
marker_dir="$(mktemp -d)"
touch "$marker_dir/compile-start"
echo "$marker_dir"
```

```bash
clojure -M:cljd compile
```

If compilation fails, restore the saved file over `deps.edn`, report the error together with any change from step 3 that explains it, and stop:

```bash
cp "${TMPDIR:-/tmp}/deps.edn.cljd-upgrade" deps.edn
```

Restoring the whole file reverts both `:tag` and `:sha`.

When the compile passes, check the generated Dart. A new release can change the Dart that ClojureDart writes, and `clojure -M:cljd compile` does not run the Dart compiler, so code that compiles can still stop `flutter build`:

1. Check that `.dart_tool/package_config.json` exists. Without it, the analyzer reports every `package:` import as `uri_does_not_exist`. When it is missing, report the check as `SKIPPED` and tell the user to run `flutter pub get`.
2. List the project's files that this compile wrote. Files under `lib/cljd-out/cljd/` hold ClojureDart's own namespaces, and files the compile did not write are output from earlier runs, so both are left out:

   ```bash
   find lib/cljd-out -name '*.dart' -newer "<marker-dir>/compile-start" -not -path 'lib/cljd-out/cljd/*'
   ```

3. Pass the listed files to the analyzer. `analysis_options.yaml` excludes `lib/cljd-out/**`, but the analyzer still analyzes a file named on the command line:

   ```bash
   dart analyze --no-fatal-warnings <listed files...>
   ```

Exit code `0` is `PASS`, and any warning lines are shown without failing. Exit code `3` is `FAIL`, as is any other non-zero exit. On `FAIL`, show every `error - <file>:<line>:<column> - <message> - <code>` line with the source namespace for each file (`lib/cljd-out/my-app/screen.dart` comes from `my-app.screen`) and any change from step 3 that explains it. The error can predate the upgrade, so ask the user whether to keep the upgrade or to restore `deps.edn` with the command above. Do not restore it without asking.

### 6. Report

Print a summary:

```
ClojureDart upgraded:
  Previous: <old-tag or "no tag"> (<old-sha>)
  Current:  <new-tag> (<new-sha>)
  Compile:  PASS
  Dart:     PASS/FAIL/SKIPPED
  Changes:  <each relevant change from step 3, or "none found">
```

## Gotchas

- Do not pin `tensegritics/clojuredart` to the `HEAD` commit from `git ls-remote`. `HEAD` can be ahead of the latest release. A coordinate without a `:tag` gets no downgrade protection, so the next `clojure -M:cljd upgrade` moves such a project back to the latest release.
