# Dart Interop, Types, and Classes

Part of the `clojuredart` skill. It applies to ClojureDart and overrides the host-neutral `clojure` skill in this domain.

## Dart Interop Syntax

### Property Access

```clojure
;; Read: (.-prop obj)
(.-length my-list)
(.-isNotEmpty my-string)

;; Set: (.-prop! obj value) or (set! (.-prop obj) value)
(.-title! app-bar "New Title")
(set! (.-text controller) "hello")

;; Chained static properties
m/Colors.purple.shade900
;; Can terminate with a method call
(m/Colors.purple.shade900.withAlpha 128)
```

### Method Calls

```clojure
;; (.method object args...)
(.contains url "example")
(.format formatter date)
(.replaceAll s ".00" "")

;; Named parameters: .paramName value
(.firstWhere list predicate .orElse fallback-fn)
(m/Text "Hello world" .maxLines 2 .softWrap true .overflow m/TextOverflow.fade)
```

### Constructors

ClojureDart constructors omit the trailing dot and `new` (both are accepted but optional, see Key Rule 6 in `SKILL.md`). Classes in function position call the default constructor:

```clojure
;; Default constructor: class in function position
(StringBuffer "hello")
(m/TextStyle .color m/Colors.red .fontSize 32.0)
(m/ThemeData .primarySwatch m.Colors/pink)

;; Named constructor: called like static methods
(List/empty .growable true)
(DateTime/now)
(tz/TZDateTime.from date-time city-tz)
```

### Static Methods and Properties

```clojure
;; Static method
(DateTime/parse "2024-01-01")
(tz/getLocation "America/New_York")

;; Static properties (including enums)
m/TextAlign.left
m/Colors.red
m/TextOverflow.fade
```

### Extension Methods

Extension methods require explicit qualification with the extension type:

```clojure
;; Dart: now.copyWith(year: now.year - 1)
;; ClojureDart:
(let [now (DateTime/now)]
  (-> now dart:core/DateTimeCopyWith (.copyWith .year (dec (.-year now)))))
```

## Types

### Type Hints

```clojure
;; Parameter type hints
(defn my-fn [^String s ^int n] ...)

;; Type hints on intermediate values
(.replaceAll ^String r pattern replacement)
```

### Nullability

`^String` means the value cannot be nil, which differs from JVM Clojure, where a hinted value can still be nil (Iron Law CLJD-3). Passing nil to a `^String` parameter compiles without a warning and throws a type error at runtime. Use `^String?` to allow nil:

```clojure
(defn greet [^String name]    ...)   ;; name is never nil
(defn greet [^String? name]   ...)   ;; name can be nil
```

### Parametrized Types

Dart generics exist at runtime. Use `#/(Type Params)` for type hints and constructors:

```clojure
;; As type hints
^#/(List String) items            ;; List<String>
^#/(Map String int) counts        ;; Map<String, int>

;; As parametrized constructors
(#/(m/GlobalKey m/FormState))                          ;; GlobalKey<FormState>()
(#/(m/MaterialPageRoute Object) .builder (f/build w))  ;; MaterialPageRoute<Object>(builder: ...)
```

### Dynamic Warnings

Dynamic warnings are similar to reflection warnings in Clojure but more serious. They occur when the compiler cannot infer the type of an object for a method call. Unlike JVM Clojure, Dart lacks reflective features, so dynamic calls:

- Are slower
- Can fail at runtime because arguments are not properly cast

Fix dynamic warnings immediately, starting with the first one, because one unresolved type propagates to downstream calls. Every warning reads `DYNAMIC WARNING: can't resolve member <name> on target type <type> of library <library>`. When the target type is `dynamic`, a type hint fixes it. When the target type is a real type, the member name or the target's type is wrong. The Gotchas section of `SKILL.md` describes both. To make a namespace's warnings compile errors (Iron Law CLJD-2), add `:no-dynamic true` to its attribute map:

```clojure
(ns my.app.core
  "No dynamic calls allowed here."
  {:no-dynamic true})
```

## Visibility and Export

```clojure
(defn ^:export my-function [args] ...)  ;; Dart-visible
(defn- helper-function [args] ...)      ;; Namespace-private
```

## Object Destructuring

### :flds for Named Fields

`:flds` destructures Dart object properties, similar to `:keys` for maps:

```clojure
(let [{:flds [height width]} size]
  (* height width))

;; Equivalent to:
(let [height (.-height size)
      width  (.-width size)]
  (* height width))
```

### .-property Keys in Destructuring

Use `.-property` as destructuring keys to extract Dart properties inline. Common in event handler parameters:

```clojure
;; Extract localPosition from DragStartDetails
(fn [{local-pos .-localPosition :as ^g/DragStartDetails details}] ...)

;; Nested: extract dx/dy from localPosition
(fn [^m/DragUpdateDetails {{:flds [dx dy]} .-localPosition}] ...)

;; Deep chain through :get
:get {{{:flds [displayLarge]} .-textTheme} m/Theme}
```

## Function Parameters

### Named Parameters

Use `.paramName` for Dart named parameters in function definitions:

```clojure
[a b c .d .e]         ;; 3 positional, 2 optional named (d, e)
[.a 42 .e]            ;; 2 named: a defaults to 42, e is required
```

### Optional Positional Parameters

Use `...` to mark the boundary between fixed and optional positional parameters:

```clojure
[a b c ... d e]       ;; 3 fixed positional, 2 optional positional
[... a 42 b]          ;; 2 optional positional: a defaults to 42
```

## Consts

ClojureDart maximally infers `const` expressions (Dart compile-time constants). In rare cases where you need a unique instance (like a sentinel), tag with `^:unique`:

```clojure
^:unique (Object)   ;; Guaranteed unique instance, not deduplicated
```

## Creating Classes

`reify`, `deftype`, and `defrecord` support Dart class features:

### Extending Classes

Build widgets with `f/widget` instead of extending `StatelessWidget` or `StatefulWidget`. Use `:extends` for Flutter classes that `f/widget` does not replace, such as `CustomPainter` or `ChangeNotifier`:

```clojure
(deftype CirclePainter []
  :extends (m/CustomPainter)
  (paint [this canvas size]
    (.drawCircle canvas (.center size m/Offset.zero) 20.0 (m/Paint)))
  (shouldRepaint [this old-delegate] false))
```

### Mixins

Tag mixin types with `^:mixin`:

```clojure
(deftype MyClass []
  :extends SomeBase
  ^:mixin SomeMixin
  ...)
```

### Operator Overloading

Use strings for operators that are not valid Clojure symbols:

```clojure
(. list "[]=" i 42)   ;; list[i] = 42
```

For operators that are valid Clojure symbols, use dot-method syntax:

```clojure
(.- a b)    ;; a - b (subtraction, e.g. Offset subtraction)
(.+ a b)    ;; a + b (addition)
```

Note: `(.- obj)` with one argument is property access. `(.- a b)` with two arguments is the `-` operator.

### Getters and Setters

Defined as methods in `reify`/`deftype`/`defrecord`. A getter takes `[this]`, a setter takes `[this v]`. New properties need `^:getter` or `^:setter` tags:

```clojure
(deftype MyType [^:mut _value]
  (^:getter value [this] _value)
  (^:setter value [this v] (set! _value v)))
```

## Imperative Dart Object Setup

Use `doto` with `set!` to configure mutable Dart objects:

```clojure
(doto (m/Paint)
  (-> .-color (set! m/Colors.grey))
  (-> .-style (set! m/PaintingStyle.fill)))
```
