---
name: cljd-upgrade
description: >-
  Upgrade ClojureDart to the latest version. Use when the user asks to upgrade,
  update, or bump ClojureDart, deps.edn SHA, or tensegritics/clojuredart dependency.
user-invocable: true
disable-model-invocation: true
---

# Upgrade ClojureDart

Upgrade the ClojureDart dependency in `deps.edn` to the latest commit and verify the project compiles.

## Prerequisites

Verify `deps.edn` exists in the current directory and contains a `tensegritics/clojuredart` dependency. If not, stop and tell the user.

## Steps

### 1. Record Current SHA

Read `deps.edn` and note the current `:sha` value for `tensegritics/clojuredart`.

### 2. Upgrade

Run the built-in upgrade command:

```bash
clj -M:cljd upgrade
```

### 3. Check for Changes

Read `deps.edn` again and compare the `:sha` value to the recorded value.

If the SHA changed, continue to step 4.

If the SHA did not change, the built-in command may have a stale cache. Fetch the latest SHA directly:

```bash
git ls-remote https://github.com/tensegritics/ClojureDart.git HEAD
```

If the remote SHA differs from `deps.edn`, update the `:sha` value in `deps.edn` to the remote SHA. If they match, report "Already at the latest version" and stop.

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
