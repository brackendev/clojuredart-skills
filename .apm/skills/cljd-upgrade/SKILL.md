---
name: cljd-upgrade
description: Upgrade the tensegritics/clojuredart SHA in deps.edn and verify the project compiles
argument-hint: "[--report] [all]"
user-invocable: true
disable-model-invocation: true
---

# Upgrade ClojureDart

Upgrade the ClojureDart dependency in `deps.edn` to the latest commit and verify the project compiles. See `CONVENTIONS.md` in the repo root for the argument grammar this skill follows.

## Arguments

| Input         | Target                                                                       |
|---------------|------------------------------------------------------------------------------|
| (no argument) | Upgrade `tensegritics/clojuredart` `:sha` in `deps.edn` and run `clj -M:cljd compile` to verify |
| `all`         | Same as (no argument); accepted for family consistency                       |
| `--report`    | Print the current `:sha` and the remote `HEAD` from `git ls-remote` without writing `deps.edn` or running the compile |

The skill rewrites a single line in `deps.edn`. `<path>` rows are not part of the standard scope vocabulary for this skill because the target is fixed.

## Mutation

Mutates `deps.edn` by default, replacing the `:sha` value under `tensegritics/clojuredart`. Also runs `clj -M:cljd compile`, which writes generated Dart under `lib/cljd-out/` (build artifact). On compile failure, the skill reverts `:sha` to the recorded original value. With `--report`, the skill writes nothing and runs no compile.

## Prerequisites

Verify `deps.edn` exists in the current directory and contains a `tensegritics/clojuredart` dependency. If not, stop and tell the user.

## Steps

Parse `$ARGUMENTS` to determine whether `--report` is present.

### 1. Record Current SHA

Read `deps.edn` and note the current `:sha` value for `tensegritics/clojuredart`.

### 2. Resolve Remote SHA

Fetch the latest SHA from GitHub:

```bash
git ls-remote https://github.com/tensegritics/ClojureDart.git HEAD
```

If `--report` is present, print the previous and remote SHAs and stop without writing `deps.edn` or running the compile:

```
ClojureDart upgrade report:
  Current: <old-sha>
  Remote:  <remote-sha>
```

### 3. Upgrade

Run the built-in upgrade command:

```bash
clj -M:cljd upgrade
```

Read `deps.edn` again and compare the `:sha` value to the recorded value. If the SHA changed, continue to step 4. If the built-in command left the SHA unchanged but the remote SHA differs, update the `:sha` value in `deps.edn` to the remote SHA. If the current and remote SHAs match, report "Already at the latest version" and stop.

### 4. Compile

```bash
clj -M:cljd compile
```

If compilation fails, revert the `:sha` in `deps.edn` to the original value, report the error, and stop.

### 5. Report

Print a summary:

```
ClojureDart upgraded:
  Previous: <old-sha>
  Current:  <new-sha>
  Compile:  PASS
```
