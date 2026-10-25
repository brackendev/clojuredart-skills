# Namespaces, Entry Points, and Dart Packages

Part of the `clojuredart` skill. It applies to ClojureDart and overrides the host-neutral `clojure` skill in this domain.

## Namespace Conventions

```clojure
(ns my-app.feature
  (:require
   ;; Dart package imports: string + :as
   ["package:flutter/material.dart" :as m]
   ["package:intl/intl.dart" :as intl]
   [cljd.flutter :as f]

   ;; App-internal Dart imports
   ["package:my_app/constants.dart" :as constants]

   ;; Other .cljd files: Clojure-style requires
   my-app.extensions.string))
```

Dart package names in require strings use underscores. ClojureDart namespace segments use hyphens. File `src/my_app/feature.cljd` maps to namespace `my-app.feature`.

## Entry Point Patterns

### Delegating to Dart main (hybrid project)

```clojure
(ns my-app.main
  (:require
   ["package:my_app/main.dart" :as dart-main]
   my-app.extensions.string))

(defn ^:export main []
  (dart-main/main))
```

### Pure ClojureDart main

```clojure
(ns my-app.main
  (:require
   ["package:flutter/material.dart" :as m]
   [cljd.flutter :as f]))

(defn main []
  (f/run
    (m/MaterialApp .title "My App")
    .home
    (m/Scaffold .appBar (m/AppBar .title (m/Text "My App")))
    .body
    m/Center
    (m/Text "Hello from ClojureDart!")))
```

## Adding Dart Packages

Use `flutter pub add` to add Dart/Flutter packages, then require them as strings in ClojureDart:

```bash
flutter pub add go_router
flutter pub add http
flutter pub add shared_preferences
```

```clojure
(ns my-app.feature
  (:require
   ["package:go_router/go_router.dart" :as go]
   ["package:http/http.dart" :as http]
   ["package:shared_preferences/shared_preferences.dart" :as prefs]))
```

Copy the exact require string from the package's Dart import statement on pub.dev. Namespace Conventions covers the underscore rule for package names.

For dev-only dependencies (testing, linting):

```bash
flutter pub add --dev integration_test
```

## Platform Channels and FFI

### Platform Views

Embed native platform views with `UiKitView` (iOS) or `AndroidView`:

```clojure
(m/UiKitView
  .viewType "my-native-view"
  .creationParams {:config "value"}
  .creationParamsCodec (m/StandardMessageCodec))
```

### dart:ffi

Access native C libraries through Dart's FFI:

```clojure
(ns my-app.native
  (:require ["dart:ffi" :as ffi]))

(let [lib (ffi/DynamicLibrary.open "libmy_native.so")
      fn-ptr (.lookup lib #/(ffi/NativeFunction (-> ffi/Int32 ffi/Int32)) "my_function")
      my-fn (.asFunction fn-ptr #/(-> int int))]
  (my-fn 42))
```

Generic type parameters in FFI use the `#/(Type Params)` tagged literal syntax. The function signature types (`ffi/Int32`, `ffi/Void`) describe the C ABI; the `.asFunction` types describe the Dart types.

## Conditional Reading

ClojureDart uses the Clojure reader, so `:clj` is always on. Put `:clj` last in reader conditionals. For macros needing Clojure host code, use `:cljd/clj-host`:

```clojure
#?(:cljd (dart-specific-code)
   :clj  (clojure-fallback))
```
