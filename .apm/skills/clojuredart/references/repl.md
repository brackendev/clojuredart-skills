# REPL

Part of the `clojuredart` skill. It applies to ClojureDart and overrides the host-neutral `clojure` skill in this domain.

## REPL

The ClojureDart README lists the REPL as "beta, clojure.repl (`doc`, `apropos`...) not ported." Expect the limitations below, and fall back to editing files and hot reload when the REPL misbehaves.

After running `clojure -M:cljd flutter`, a socket REPL starts. Once the app reports that it is ready, the watcher announces the port. In a terminal the line reads `🤫 ClojureDart REPL listening on port <port>`. When the output is not a terminal, as when an agent runs the command, it reads:

```
=== 🤫 ClojureDart REPL === listening on port === <port> ===
```

The port varies between runs, so read it from this line. The watcher also writes the port and its process ID to `REPL.lock` in the project root. The examples below use `<port>` for it.

Connect with:

```bash
nc localhost <port>
```

Or from Emacs: `C-u M-x inferior-lisp` and enter `nc localhost <port>`.

If the banner does not appear, the app has not reported ready (it is still building, it failed, or the target is web), or the command is not `clojure -M:cljd flutter`. `clojure -M:cljd watch` also writes `REPL.lock` but runs no app, so it never prints the banner. While `clojure -M:cljd flutter` or `watch` runs, `REPL.lock` makes every other `clojure -M:cljd compile`, `watch`, `test`, or `flutter` in the project exit with `Another ClojureDart process is running (PID <pid>)`. REPL Limitations below lists the supported targets.

Forms evaluate in the running Dart isolate through hot reload, so evaluation can lag. The default namespace is `cljd.user`, which refers `pick!` and `mount!` (from `cljd.flutter.repl`) and aliases `cljd.flutter` as `f` and `package:flutter/material.dart` as `m`. Special vars: `*1`, `*2`, `*3`, `*e`, and `*env` (bound after `pick!`). Switch namespace with `(ns my.app.core)`. When the namespace already exists, the `ns` form only switches to it and ignores its `:require` clauses, so the namespace's existing aliases stay available.

### Driving Live App State

Because `:watch` subscribes widgets to atoms, swapping a state atom from the REPL re-renders the UI with no file edit:

```clojure
;; In the REPL, fully-qualified names work from cljd.user without requires:
(swap! my.app.state/auth-state assoc :error "Hello from REPL")
;; UI updates immediately

(swap! my.app.state/auth-state assoc :error nil)
;; UI reverts
```

### Interactive Widget Inspection

```clojure
;; Toggle the widget picker, then tap a widget in the running app
(pick!)

;; After picking, *env maps :ctx (the BuildContext), :source-loc, and
;; :full-env, plus one symbol key per lexical binding at the widget
(keys *env)
;; => (:ctx :source-loc :full-env counter ...)

(get *env 'counter)
;; => #object[cljd.core.Atom ...]

;; Replace the picked widget; the replaced widget becomes *1
(mount!
  (f/widget
    :watch [n (get *env 'counter)]
    (m/Text (str "Debug: " n))))

;; Restore the original widget
(mount! nil)
```

### Driving the REPL from a Script

For scripted use (tests, automation, agent tool calls), send every form through one connection, and close that connection by stopping `nc` while its input is still open. When `nc` reaches the end of its input first, it closes only its sending side, and the REPL session then answers `Error:  Write end dead` over and over, triggering a hot reload for each, until the client exits. A trailing `sleep` longer than the `timeout` (GNU coreutils) keeps the input open:

```bash
# Multi-form session with interleaved external actions.
(
  echo '(swap! my.app.state/auth-state assoc :error "Hello")'
  sleep 2
  # Take a screenshot, run assertions, etc.
  echo '(swap! my.app.state/auth-state assoc :error nil)'
  sleep 60
) | timeout 10 nc localhost <port>
```

Ending a connection can stop the REPL for the rest of the run. With release `0.9.20261002`, closing an `nc` client made `clojure -M:cljd flutter` print `Exception in thread "Thread-..." java.net.SocketException: Broken pipe`, and later connections received no prompt. When a new connection prints no `cljd.user=>` prompt, restart `clojure -M:cljd flutter`.

### REPL Limitations

- **Native Dart VM only.** `clojure -M:cljd flutter -d chrome` runs the app and writes `REPL.lock`, but the watcher never sees the app's ready message and prints no REPL banner. Use `-d <ios-sim-udid>`, `-d emulator-5554`, or a desktop target such as `-d macos`.
- Socket REPL, not nREPL. Editor integration is limited; `nc` or `telnet` works.
- No `doc` or `apropos`. Use the ClojureDart source or this skill for API reference.
- REPL expressions run on the main isolate. Long-running evaluations block the UI.
- Hot reload preserves app state. A hot restart (RETURN in the watcher) prints `;;;; App restarting. Abandon all state!` to connected REPLs, and REPL `def` evaluations do not survive it.
