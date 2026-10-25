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

Mark functions as async with `^:async` when they need to return a `Future`:

```clojure
(defn ^:async fetch-data [^String url]
  (let [response (await (http/get (Uri/parse url)))]
    (.-body response)))
```

### Streams

Dart Streams are watchable in `:watch` directives. Use `:default` for the initial value before the stream emits:

```clojure
(f/widget
  :watch [position (geolocation/getPositionStream) :default nil]
  (if position
    (m/Text (str "Lat: " (.-latitude position)))
    (m/Text "Waiting for GPS...")))
```

### Isolates

Use `cljd.dart.isolates/spawn!` for background computation. Returns a map with `:in` and `:out` ports:

```clojure
(ns my-app.background
  (:require [cljd.dart.isolates :as isolates]))

(let [{:keys [in out]} (await (isolates/spawn!
                                (fn [{:keys [in out]}]
                                  ;; Runs in separate isolate
                                  (let [msg (await (.first in))]
                                    (await (.add out (process msg)))))))]
  (await (.add in data))
  (let [result (await (.first out))]
    result))
```

Use isolates for CPU-intensive work (parsing large datasets, image processing) to avoid blocking the UI thread.
