# REPL

Part of the `clojuredart` skill. It applies to ClojureDart and overrides the host-neutral `clojure` skill in this domain.

## REPL

After running `clj -M:cljd flutter`, a socket REPL starts and announces itself:

```
=== 🤫 ClojureDart REPL === listening on port === <port> ===
```

The port varies between runs, so read it from this line. The examples below use `<port>` for it.

Connect with:

```bash
nc localhost <port>
```

Or from Emacs: `C-u M-x inferior-lisp` and enter `nc localhost <port>`.

If the banner did not appear or the port file is missing, the build process is not running or the target is web. Restart `clj -M:cljd flutter` against a native Dart target. REPL Limitations below lists the targets and explains why an ordinary `nc` disconnect does not require a restart.

Forms evaluate in the running Dart isolate. The default namespace is `cljd.user`. `pick!` and `mount!` (from `cljd.flutter.repl`) are referred by default. Special vars: `*1`, `*2`, `*3`, `*e`, and `*env` (bound after `pick!`). Switch namespace with `(ns my.app.core)`.

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
;; Pick a widget on screen (tap to select in the running app)
(pick!)

;; After picking, *env contains the widget's lexical bindings
(keys *env)
;; => (:counter :ctx ...)

(get *env :counter)
;; => #<Atom@...>

;; Mount a new widget into the picked location
(mount!
  (f/widget
    :watch [n (get *env :counter)]
    (m/Text (str "Debug: " n))))
```

### Driving the REPL from a Script

Interactive use with `nc` works as documented. For scripted use (tests, automation, LLM tool calls), use a subshell so one connection sends multiple forms and can interleave external side effects (screenshots, assertions) between them.

```bash
# Multi-form session with interleaved external actions.
(
  echo '(swap! my.app.state/auth-state assoc :error "Hello")'
  sleep 2
  # Take a screenshot, run assertions, etc.
  echo '(swap! my.app.state/auth-state assoc :error nil)'
  sleep 1
) | nc localhost <port>

# Single form works too. Give the REPL a moment to write back before stdin EOFs.
(echo '(+ 1 2)'; sleep 1) | nc localhost <port>
```

### REPL Limitations

- **Native Dart VM only.** `clj -M:cljd flutter -d chrome` compiles to JS and does not print a REPL port. Use `-d <ios-sim-udid>`, `-d emulator-5554`, or a desktop target.
- **Beta: client disconnect floods `clj -M:cljd flutter` logs with `Error: Write end dead`.** Cosmetic noise after `nc` closes stdin. The form that was sent evaluates and returns to the client before the flood. A fresh `nc` connection still evaluates forms and receives responses; no restart of `clj -M:cljd flutter` is needed.
- Socket REPL, not nREPL. Editor integration is limited; `nc` or `telnet` works.
- No `doc` or `apropos`. Use the ClojureDart source or this skill for API reference.
- REPL expressions run on the main isolate. Long-running evaluations block the UI.
- Hot reload preserves app state. REPL `def` evaluations do not persist across restarts.
