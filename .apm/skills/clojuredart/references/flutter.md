# cljd.flutter

Part of the `clojuredart` skill. It applies to ClojureDart and overrides the host-neutral `clojure` skill in this domain.

## cljd.flutter

`cljd.flutter` is a utility library that removes Flutter boilerplate. Require it in every namespace that builds widgets:

```clojure
(ns my-app.main
  (:require
   [cljd.flutter :as f]
   ["package:flutter/material.dart" :as m]))
```

### f/run

Starting point for a Flutter application. Called from `main`:

```clojure
(defn main []
  (f/run
    (m/MaterialApp .title "My App")
    .home
    (m/Scaffold .appBar (m/AppBar .title (m/Text "Hello")))
    .body
    m/Center
    (m/Text "Let's get coding!")))
```

### f/build

Creates a builder callback (a function that returns a widget). Used for Flutter APIs that expect builder functions like `itemBuilder`, `builder`, etc.:

```clojure
;; Simple builder (no params) -- wraps a widget as a builder function
(#/(m/MaterialPageRoute Object) .builder (f/build second-route))

;; Builder with index parameter (for itemBuilder)
(m/ListView.builder
  .itemCount 25
  .itemBuilder (f/build [i] (fake-item (odd? i))))

;; Builder with directives
(f/build
  :get [m/Navigator]
  (m/AlertDialog .content (m/Text "hello")))
```

### f/widget

Evaluates to a Widget. Its body is interleaved expressions and directives. Expressions are `.child`-threaded by default:

```clojure
;; Expressions thread through .child
(f/widget
  m/Center
  (m/Text "hello"))
;; Equivalent to: (m/Center .child (m/Text "hello"))

;; Use dotted symbols to thread through other named params
(f/widget
  m/MaterialApp
  .home
  m/Scaffold
  .body
  m/Center
  (m/Text "hello"))
```

### Directives

Directives are keywords followed by a form, used inside `f/widget` and `f/run`:

#### :let

```clojure
(f/widget
  :let [name "World"]
  (m/Text (str "Hello " name)))
```

#### :watch (reactive state)

Watches atoms, Futures, Streams, Listenables, and other watchables. Rebuilds when values change:

```clojure
(f/widget
  :watch [v an-atom]
  (m/Text (str v)))
```

Options:

```clojure
;; :default -- initial value before async watchable produces
:watch [v (Future/delayed (Duration .seconds 3) (fn [] "Done"))
        :default "Loading..."]

;; :as -- name the watchable for mutation
:watch [n (atom 0) :as counter]
;; Then use: (swap! counter inc)

;; :dispose -- custom cleanup (defaults to nil, unlike :managed which defaults to .dispose)
:watch [v some-resource :as r :dispose .close]

;; :refresh-on -- control when watchable is recomputed (nil = never refresh)
:watch [data (atom initial-value) :as state :refresh-on nil]

;; :> -- extract value from notifier-style watchables
:watch [v some-notifier :> .-value]
```

Deduplication: `:watch` only triggers rebuild when bound values change (by `=`). Scoped Watches below shows how destructuring narrows rebuilds.

#### :managed (lifecycle management)

Automatic lifecycle for controllers and disposable objects. Calls `.dispose` by default:

```clojure
(f/widget
  :managed [controller (m/TextEditingController)]
  (m/TextField .controller controller))
```

#### :bind and :get (inherited bindings)

Dynamic binding along the widget tree (inherited widgets):

```clojure
;; Establish binding
(f/widget
  :bind {:counters (atom {:left 0 :right 0})}
  child-widget)

;; Retrieve binding -- vector form (simple)
(f/widget
  :get [:counters]
  :watch [n counters]
  (m/Text (str n)))

;; Retrieve Flutter framework objects
(f/widget
  :get [m/Navigator]
  ;; navigator is bound (auto kebab-cased from Navigator)
  (.push navigator some-route))

;; Map form -- combines framework objects, :value-of, and destructuring
(f/widget
  :get {{{:flds [displayLarge]} .-textTheme} m/Theme
        :value-of [:counters]}
  :watch [{n :left} counters]
  (m/Text (str n) .style displayLarge))

;; Nested destructuring with property access
(f/widget
  :get {{{:flds [secondary onSecondary]} .-colorScheme} m/Theme}
  (m/Material .color secondary))
```

#### :summon (inherited lookups, preferred for new code)

Upstream ClojureDart recommends `:summon` over `:get` for new code and keeps `:get` for compatibility. `:summon` takes a bindings vector like `:let`. Each right-hand side is resolved on every build, and the body is re-evaluated only when the bound or destructured value changes. Otherwise the previously built child is reused.

```clojure
;; Class: resolved through its .of method -- (m/Theme.of context)
(f/widget
  :summon [{:flds [brightness]} m/Theme]
  (m/Text (str brightness)))

;; Form with arguments: extra arguments follow the context
;; -- (m/Navigator.of context .rootNavigator true)
(f/widget
  :summon [root-navigator (m/Navigator .rootNavigator true)]
  (m/TextButton .onPressed (fn [] (.pop root-navigator)) .child (m/Text "Close")))

;; Keyword: a value established by :bind
(f/widget
  :summon [counters :counters]
  :watch [n counters]
  (m/Text (str n)))
```

A class name that already ends in `.of` or `.<name>Of` (for example, `m/Navigator.maybeOf`) is called as written.

Supported options are `:as` (bind the raw right-hand side value) and `:>` or `:value>` (equivalent: transform the value before binding or destructuring). `:summon` does not support `:default`, `:refresh-on`, `:dedup`, or disposal options; use `:watch` or `:managed` for those.

When the binding position is a keyword, `:of` and `:keys` are shorthand:

```clojure
:summon [:of [m/checkbox-theme]
         :keys [data/user]]

;; is equivalent to:
:summon [checkbox-theme m/CheckboxTheme
         user :data/user]
```

The upstream clj-kondo hooks do not recognize `:summon` yet (see Gotchas in `SKILL.md`). Existing `:get` code keeps working, and the examples in this package still use `:get`.

#### :context

Binds a `BuildContext` instance:

```clojure
(f/widget
  :context ctx
  (m/showDialog .context ctx ...))
```

#### :bg-watcher (background side effects)

Runs side effects when a watchable changes without triggering widget rebuild:

```clojure
(f/widget
  :managed [animator (m/AnimationController .vsync vsync)]
  :watch [{:keys [alignment animation]} state]
  :bg-watcher ([animated-value animation]
               (swap! state assoc :alignment animated-value))
  ...)
```

#### :vsync

Binds a `TickerProvider` for animations:

```clojure
(f/widget
  :vsync clock
  :managed [controller (m/AnimationController .vsync clock .duration (Duration .seconds 1))]
  ...)
```

#### :key

Wraps its value in a `ValueKey` for identifying sibling widgets:

```clojure
(f/widget
  :key item-id
  (m/ListTile .title (m/Text item-name)))
```

#### :when

Conditionally shows the rest of the widget:

```clojure
(f/widget
  :when show?
  (m/Text "Visible"))
```

#### :height, :width, :color

Shorthand for `SizedBox` and `ColoredBox`:

```clojure
(f/widget
  :height 100
  :width 200
  :color m/Colors.blue
  (m/Text "Sized and colored"))
```

#### :padding

```clojure
:padding 16                           ;; EdgeInsets.all(16)
:padding {:horizontal 8 :vertical 4}  ;; EdgeInsets.symmetric
:padding {:top 8 :bottom 16}          ;; EdgeInsets.only
```

### Cells (Derived State)

`f/$` creates a cell that recomputes when dependencies change (like a spreadsheet cell). Dependencies are read with `f/<!`:

```clojure
(let [total (f/$ (reduce + (f/<! items-atom)))]
  ...)
```

Cells can depend on other cells and atoms. The dependency graph updates automatically:

```clojure
(def items (atom [{:name "A" :price 10} {:name "B" :price 20}]))

;; Derived: recomputes when items changes
(def total (f/$ (reduce + (map :price (f/<! items)))))

;; Derived from derived: recomputes when total changes
(def tax (f/$ (* 0.1 (f/<! total))))
```

Share cells via `:bind`/`:get` rather than passing as function arguments:

```clojure
;; Root: bind the atom and derived cells
(f/widget
  :bind {:cart (atom [])
         :cart-total (f/$ (reduce + (map :price (f/<! (:cart %)))))}
  (shop-screen))

;; Consumer: get the derived cell
(f/widget
  :get [:cart-total]
  :watch [total cart-total]
  (m/Text (str "Total: $" total)))
```

### Scoped Watches

`:watch` triggers rebuilds when the watched value changes (by `=`). For large state atoms, destructure to watch only the fields you need. This prevents rebuilds when unrelated fields change:

```clojure
;; Rebuilds on any change to app-state (wasteful)
(f/widget
  :watch [state app-state]
  (m/Text (:name state)))

;; Rebuilds only when :name changes (scoped)
(f/widget
  :watch [{:keys [name]} app-state]
  (m/Text name))
```

For even finer control, use cells to pre-compute the exact value a widget needs:

```clojure
(def selected-name (f/$ (:name (f/<! app-state))))

(f/widget
  :watch [name selected-name]
  (m/Text name))
```

### Choosing f/widget vs f/build

| Need | Use |
|------|-----|
| A widget to place in the tree | `f/widget` |
| A callback for a `builder:` parameter | `f/build` |
| A route builder for go_router or MaterialPageRoute | `f/build` |
| An `itemBuilder` for ListView.builder | `f/build` with index parameter |

`f/widget` evaluates to a `Widget` instance. `f/build` evaluates to a function that returns a `Widget`. Flutter APIs that accept `builder:`, `itemBuilder:`, or route builder parameters expect functions, not widgets.

### Choosing Directives

| Need | Use |
|------|-----|
| Local mutable state within a widget | `atom` inside `:watch` with `:as` |
| Shared state across distant widgets | `atom` in `:bind`/`:get` |
| Inherited Flutter object (`Theme`, `Navigator`, `MediaQuery`) or a `:bind` value, rebuilding only when it changes | `:summon` (or `:get` in existing code) |
| Derived value that recomputes when dependencies change | `f/$` cell with `f/<!` |
| Dart controller with lifecycle (dispose) | `:managed` |
| React to external async source (Future, Stream, Listenable) | `:watch` with `:default` |
| Side effect on state change (animation, logging, persistence) without triggering rebuild | `:bg-watcher` |
