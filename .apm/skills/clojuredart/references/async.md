# Async, Streams, and Isolates

Part of the `clojuredart` skill. It applies to ClojureDart and overrides the host-neutral `clojure` skill in this domain.

## Async

### await

Use `await` for Dart async operations. A Dart method that returns a `Future` must be awaited. Without `await`, the expression evaluates to the `Future` object instead of the resolved value:

```clojure
(let [result (await (some-async-call))]
  (process result))
```

### Async Functions

A function whose body uses `await` compiles to a Dart `async` function without any tag. Mark a function `^:async` when it must return a `Future` but does not await anything itself:

```clojure
(defn fetch-data [^String url]
  (let [response (await (http/get (Uri/parse url)))]
    (.-body response)))
```

### Streams

Dart Streams are watchable in `:watch` directives. Until the stream emits, the binding is `nil`, or the value given with `:default`:

```clojure
(f/widget
  :watch [n (Stream/periodic (Duration .seconds 1) (fn [i] i)) :default 0]
  (m/Text (str "Ticks: " n)))
```

### Isolates

Use `cljd.dart.isolates/spawn!` for background computation. It returns a `Future` of a map whose `:ports` entry holds the ports. By default the caller gets `:in` as a `SendPort` and `:out` as a `ReceivePort`, and the spawned function receives the matching ends: `:in` as a `ReceivePort` and `:out` as a `SendPort`. Send with `.send`, and read one message with `.-first`:

```clojure
(ns my-app.background
  (:require ["dart:isolate" :as dart:isolate]
            [cljd.dart.isolates :as isolates]))

(defn double-in-isolate [n]
  (let [{{:keys [in out]} :ports}
        (await (isolates/spawn!
                 (fn [{:keys [in out]}]
                   ;; Runs in the new isolate
                   (let [msg (await (.-first ^dart:isolate/ReceivePort in))]
                     (.send ^dart:isolate/SendPort out (* 2 msg))))))]
    (.send ^dart:isolate/SendPort in n)
    (await (.-first ^dart:isolate/ReceivePort out))))
```

Use isolates for CPU-intensive work (parsing large datasets, image processing) to avoid blocking the UI thread.
