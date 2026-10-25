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

   ;; App-internal Dart imports use the package name from pubspec.yaml
   ["package:cljd_my_app/constants.dart" :as constants]

   ;; Other .cljd files: Clojure-style requires
   my-app.extensions.string))
```

Dart package names in require strings use underscores. ClojureDart namespace segments use hyphens. File `src/my_app/feature.cljd` maps to namespace `my-app.feature`.

`clojure -M:cljd init` derives the Dart package name from the project directory: it lowercases the name, replaces every character outside `a-z`, `0-9`, and `_` with `_`, and adds the prefix `cljd_`. A directory named `my_app` becomes the package `cljd_my_app`, so app-internal imports read `package:cljd_my_app/...`. Read the `name:` line in `pubspec.yaml` before writing one.

## Entry Point Patterns

### Delegating to Dart main (hybrid project)

`clojure -M:cljd init` writes `lib/main.dart` as a one-line export of the ClojureDart `main`, so keep the Dart entry point in another file, such as `lib/app_main.dart`:

```clojure
(ns my-app.main
  (:require
   ["package:cljd_my_app/app_main.dart" :as dart-main]
   my-app.extensions.string))

(defn main []
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

For dev-only dependencies (testing, linting), use `flutter pub add --dev <package>`. Packages that ship with the Flutter SDK, such as `integration_test`, need the SDK form from the Flutter integration test docs, because `flutter pub add --dev integration_test` resolves a pub.dev package and fails version solving:

```bash
flutter pub add "dev:integration_test:{sdk: flutter}"
```

## Platform Channels and FFI

### Platform Views

Embed native platform views with `UiKitView` (iOS) or `AndroidView`. `StandardMessageCodec` lives in `package:flutter/services.dart`, which `material.dart` does not provide. The codec encodes strings, numbers, booleans, lists, and maps but not keywords, so give `creationParams` string keys:

```clojure
(ns my-app.native-view
  (:require
   ["package:flutter/material.dart" :as m]
   ["package:flutter/services.dart" :as services]))

(m/UiKitView
  .viewType "my-native-view"
  .creationParams {"config" "value"}
  .creationParamsCodec (services/StandardMessageCodec))
```

### dart:ffi

Access native C libraries through Dart's FFI:

```clojure
(ns my-app.native
  (:require ["dart:ffi" :as ffi]))

(let [lib (ffi/DynamicLibrary.open "libmy_native.so")
      my-fn (-> lib
              (. #/(lookup (ffi/NativeFunction (ffi/Int32 -> ffi/Int32))) "my_function")
              ffi/NativeFunctionPointer
              (. #/(asFunction (int -> int))))]
  (my-fn 42))
```

This compiles to Dart's `lib.lookup<NativeFunction<Int32 Function(Int32)>>("my_function").asFunction<int Function(int)>()`. A method's type arguments attach to its name with `#/(method TypeArgs)`, called through `(. obj #/(method ...) args)`. A function type is written `#/(ArgType ... -> ReturnType)`, so `(-> ffi/Void)` is `Void Function()`. `asFunction` is an extension method, so the call names its extension, `ffi/NativeFunctionPointer` (see Extension Methods in `references/dart-interop.md`). The `NativeFunction` types (`ffi/Int32`, `ffi/Void`) describe the C ABI, and the `asFunction` types describe the Dart side. Upstream's `samples/ffi` shows the same pattern.

## Conditional Reading

ClojureDart reads reader conditionals in `.cljd` and `.cljc` files. The rules for every platform (the tags, `:default`, `#?@` splicing, and which files read conditionals) are in `clojure/references/reader-conditionals.md` in the `clojure` skill (clojure-skills). This section covers what is specific to ClojureDart.

The compiler reads each file twice. The host pass runs on the JVM and evaluates the `ns` form, macros, and `^:macro-support` functions, and it reads with `:cljd`, `:cljd/clj-host`, and `:clj` all on. The Dart pass reads with only `:cljd` on. The first matching tag wins, so put `:clj` last:

```clojure
(ns my-app.paths
  (:require #?(:cljd ["dart:io" :as io]
               :clj  [clojure.java.io :as io])))
```

With `:clj` first, the host pass takes the `:clj` branch and looks for `clojure.java.io` as `cljd.java.io`, and the compile stops with `compiling namespace cljd.java.io` and the `:clj` require as the faulty form. Upstream's FAQ gives the same rule. In the body of an ordinary function, which only the Dart pass compiles, the `:clj` branch never takes effect, but keeping `:clj` last everywhere avoids the failure in the forms that the host pass evaluates.

In a macro body, which runs on the JVM, use `:cljd/clj-host` for the host code that the ClojureDart build of the macro needs. It keeps that code apart from the `:clj` branch that JVM Clojure uses. Put `:cljd/clj-host` first, because `:cljd` is also on in the host pass. Upstream `cljd/flutter/repl_impl.cljd` writes:

```clojure
#?(:cljd/clj-host (.toEpochMilli (java.time.Instant/now))
   :cljd (.-millisecondsSinceEpoch (DateTime/now)))
```
