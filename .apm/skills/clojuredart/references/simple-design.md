# Simple Design in ClojureDart

Part of the `clojuredart` skill. It applies to ClojureDart and gives the host constructs for Key Rule 20 of the host-neutral `clojure` skill. Read `clojure/references/simple-design.md` in the `clojure` skill first. Its rules and its exceptions apply here unchanged. The claims below were checked by compiling and running a Dart program with ClojureDart `0.9.20261002`.

## State: atoms, and values that arrive later

ClojureDart has atoms and no refs, agents, or `dosync`. The baseline rule holds: dereference once at the edge and pass the value inward.

- Keep related state in one atom that holds a nested map, so one dereference returns all of it.
- A Dart `Future` becomes a value once. `await` it at the edge and pass the result to pure functions (`references/async.md`). A function that takes a `Future` also takes on its waiting and its failure.
- In Flutter code, a widget reads state through `:watch`, and a cell derives a value from atoms (`references/flutter.md`). Pass the watched value to the functions that build or compute, not the atom.

## Queues between components

The baseline says to put a queue between a producer and a consumer that should not know each other, or that run at different rates or times. For sequential work, keep the direct call. ClojureDart has no `core.async`. Dart's own queue between components is a `Stream`, fed by a `StreamController` from `dart:async`.

```clojure
(ns my-app.indexing
  (:require ["dart:async" :as dart:async]))

;; good: the producer knows only the controller and the shape of the message
(defn import-record! [^dart:async/StreamController index-queue record]
  (.add index-queue {:index/op :add :index/record record}))

;; good: the consumer listens to the stream
(defn run-indexer! [^dart:async/StreamController index-queue index!]
  (.listen (.-stream index-queue)
           (fn [message] (index! (:index/record message)))))

(def ^dart:async/StreamController index-queue (dart:async/StreamController))
```

- A controller made with `(dart:async/StreamController)` allows one listener, and it keeps events that arrive before the listener subscribes. A second `.listen` throws "Bad state: Stream has already been listened to."
- A controller made with `(dart:async/StreamController.broadcast)` allows several listeners, and it discards events that arrive while nobody listens.
- A controller has no capacity argument. When the producer can be faster than the consumer, limit the work at the producer.
- Put plain data on the stream, such as maps with namespaced keys.
- Mark the end with `.close`, which returns a `Future`, and cancel the subscription that `.listen` returns when the consumer stops.
- Pass the controller, or only its stream, to each side as an argument.
- Between isolates, the ports that `cljd.dart.isolates/spawn!` returns are the queue (`references/async.md`).

## Extend protocols to Dart types

`extend-protocol` connects a protocol to a Dart type that the program does not own, so no wrapper type is needed. Name Dart types by their class names, such as `String`, `int`, and `Null`.

```clojure
(defprotocol Summary
  (summarize [x]))

(extend-protocol Summary
  String
  (summarize [s] (subs s 0 (min 20 (count s))))
  int
  (summarize [n] (str "number " n))
  Null
  (summarize [_] "nothing"))
```

- Extend to `Null` to give absence an explicit answer.
- For a catch-all, extend to the `fallback` pseudotype instead of `Object`. The ClojureDart documentation says that it has a lower priority than other extensions, and `satisfies?` returns `false` for a value that uses it. Add one only when one answer is correct for every type. Without it, an unsupported value throws "No extension of protocol Summary found for type", which reports the gap.
- Multimethods work only inside one namespace in this release. `defmulti` and `defmethod` compile and dispatch when they share a namespace, although the upstream list of differences from Clojure still names multimethods as missing. A `defmethod` for a multimethod that another namespace defines fails to compile on a fresh build ([ClojureDart pull request #377](https://github.com/Tensegritics/ClojureDart/pull/377), open). When another namespace must add a case, use a protocol: `extend-protocol` from another namespace compiles and dispatches.
- `reify`, `deftype`, and `defrecord` implement Dart classes and interfaces (`references/dart-interop.md`).
