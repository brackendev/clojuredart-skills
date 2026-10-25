---
name: clojuredart-nav
description: >-
  Use when adding or changing navigation in a ClojureDart Flutter app (.cljd
  files): named routes and `pushNamed`, go_router (`GoRouter`, `GoRoute`,
  `ShellRoute`, `MaterialApp.router`), Navigator push/pop and results,
  dialogs and bottom sheets, tab navigation (`BottomNavigationBar`, `TabBar`),
  route parameters, and deep linking.
user-invocable: false
---

# ClojureDart Navigation

Navigation patterns for ClojureDart Flutter applications.

## Choosing an Approach

| Approach | When to Use |
|----------|-------------|
| Named routes | Simple apps with a flat list of screens and no deep linking |
| go_router | Apps needing deep linking, URL-based routing, nested navigation, or web support |
| Navigator API | Programmatic navigation, modal sheets, dialogs, or custom transitions |
| Tab navigation | Bottom tabs, top tabs, or drawer-based section switching |

Named routes and go_router are declarative. Navigator API is imperative. Most apps combine declarative routing with imperative pushes for modals and dialogs.

## Named Routes

Define routes in `MaterialApp` and navigate by name:

```clojure
(ns my-app.main
  (:require
   ["package:flutter/material.dart" :as m]
   [cljd.flutter :as f]
   [my-app.screens.home :as home]
   [my-app.screens.detail :as detail]))

(defn main []
  (f/run
    (m/MaterialApp
      .initialRoute "/"
      .routes {"/"       (f/build (home/screen))
               "/detail" (f/build (detail/screen))})))
```

Navigate:

```clojure
(f/widget
  :get [m/Navigator]
  (m/ElevatedButton
    .onPressed (fn [] (.pushNamed navigator "/detail") nil)
    .child (m/Text "Go to Detail")))
```

The trailing `nil` matters: `.onPressed` expects a `void` callback, and `.pushNamed` returns a `Future`, which the Dart compiler rejects as the return value (see Key Rule 4 in the `clojuredart` skill).

Pass arguments:

```clojure
;; Push with arguments
(.pushNamed navigator "/detail" .arguments {:id 42})

;; Receive arguments in the target screen
(f/widget
  :context ctx
  :let [args (-> (m/ModalRoute.of ctx) .-settings .-arguments)]
  (m/Text (str "Item: " (:id args))))
```

## go_router

Add the package:

```bash
flutter pub add go_router
```

Define routes:

```clojure
(ns my-app.router
  (:require
   ["package:flutter/material.dart" :as m]
   ["package:go_router/go_router.dart" :as go]
   [cljd.flutter :as f]
   [my-app.screens.home :as home]
   [my-app.screens.detail :as detail]))

(def router
  (go/GoRouter
    .routes
    [(go/GoRoute
       .path "/"
       .builder (f/build [state] (home/screen)))
     (go/GoRoute
       .path "/detail/:id"
       .builder (f/build [^go/GoRouterState state]
                  (detail/screen (-> state .-pathParameters (get "id")))))]))
```

Use the router in the app:

```clojure
(ns my-app.main
  (:require
   ["package:flutter/material.dart" :as m]
   [cljd.flutter :as f]
   [my-app.router :as router]))

(defn main []
  (f/run
    (m/MaterialApp.router
      .routerConfig router/router)))
```

Navigate with go_router:

```clojure
(f/widget
  :context ctx
  (m/ElevatedButton
    .onPressed (fn [] (.go (go/GoRouter.of ctx) "/detail/42"))
    .child (m/Text "Go to Detail")))

;; Or use the extension method
(f/widget
  :context ctx
  (m/ElevatedButton
    .onPressed (fn [] (-> ctx go/GoRouterHelper (.go "/detail/42")))
    .child (m/Text "Go to Detail")))
```

### Nested Navigation (ShellRoute)

```clojure
(go/GoRouter
  .routes
  [(go/ShellRoute
     .builder (f/build [^go/GoRouterState state child]
               (my-app-shell child))
     .routes
     [(go/GoRoute .path "/" .builder (f/build [state] (home/screen)))
      (go/GoRoute .path "/settings" .builder (f/build [state] (settings/screen)))])])
```

## Navigator API

Direct imperative navigation for modals, dialogs, and custom transitions:

```clojure
;; Push a new route
(f/widget
  :get [m/Navigator]
  (m/ElevatedButton
    .onPressed
    (fn []
      (.push navigator
        (#/(m/MaterialPageRoute Object)
          .builder (f/build (detail/screen))))
      nil)
    .child (m/Text "Open Detail")))

;; Pop back
(.pop navigator)

;; Pop with result
(.pop navigator {:selected true})

;; Push and wait for result
(f/widget
  :get [m/Navigator]
  (m/ElevatedButton
    .onPressed
    (fn []
      (let [result (await (.push navigator
                            (#/(m/MaterialPageRoute Object)
                              .builder (f/build (picker/screen)))))]
        (when result
          (process-selection result))))
    .child (m/Text "Pick Item")))
```

### Dialogs and Bottom Sheets

```clojure
;; Show dialog and wait for the value passed to .pop
(f/widget
  :context ctx
  (m/ElevatedButton
    .onPressed
    (fn []
      (let [confirmed (await
                        (#/(m/showDialog bool)
                          .context ctx
                          .builder (f/build
                                     :get [m/Navigator]
                                     (m/AlertDialog
                                       .title (m/Text "Confirm")
                                       .content (m/Text "Proceed?")
                                       .actions
                                       [(m/TextButton
                                          .onPressed (fn [] (.pop navigator false))
                                          .child (m/Text "Cancel"))
                                        (m/TextButton
                                          .onPressed (fn [] (.pop navigator true))
                                          .child (m/Text "OK"))]))))]
        (when confirmed
          (proceed!))))
    .child (m/Text "Show Dialog")))

;; Show bottom sheet
(f/widget
  :context ctx
  (m/ElevatedButton
    .onPressed
    (fn []
      (m/showModalBottomSheet
        .context ctx
        .builder (f/build
                   (m/Container
                     .padding (m/EdgeInsets.all 16)
                     .child (m/Text "Bottom Sheet Content"))))
      nil)
    .child (m/Text "Show Sheet")))
```

## Tab Navigation

### BottomNavigationBar

```clojure
(defn main-screen []
  (f/widget
    :watch [idx (atom 0) :as tab-index]
    (m/Scaffold
      .body (case idx
              0 (home/screen)
              1 (search/screen)
              2 (profile/screen))
      .bottomNavigationBar
      (m/BottomNavigationBar
        .currentIndex idx
        .onTap (fn [i] (reset! tab-index i))
        .items [(m/BottomNavigationBarItem .icon (m/Icon m/Icons.home) .label "Home")
                (m/BottomNavigationBarItem .icon (m/Icon m/Icons.search) .label "Search")
                (m/BottomNavigationBarItem .icon (m/Icon m/Icons.person) .label "Profile")]))))
```

### TabBar with TabBarView

```clojure
(f/widget
  :vsync ticker
  :managed [controller (m/TabController .length 3 .vsync ticker)]
  (m/Scaffold
    .appBar (m/AppBar
              .bottom (m/TabBar
                        .controller controller
                        .tabs [(m/Tab .text "First")
                               (m/Tab .text "Second")
                               (m/Tab .text "Third")]))
    .body (m/TabBarView
            .controller controller
            .children [(first-tab) (second-tab) (third-tab)])))
```

## Gotchas

- `go_router` requires `MaterialApp.router` with `.routerConfig`, not regular `MaterialApp` with `.routes`. These are mutually exclusive.
- `f/build` in route builders receives the route state as a parameter. Type-hint it as `^go/GoRouterState` to access `.pathParameters` and `.queryParameters` without dynamic warnings.
- `Navigator.of(context)` in ClojureDart is `(m/Navigator.of ctx)`, and `Navigator.of(context).pop()` is `(.pop (m/Navigator.of ctx))`. A dotted name after the context passes a named argument to `.of`, so `(m/Navigator.of ctx .pop false)` fails to compile with `Not an expected argument name: .pop`. The same applies to `(go/GoRouter.of ctx)`. The `:get [m/Navigator]` directive is a cleaner alternative that avoids passing context manually.
- `:get [m/Navigator]` and `Navigator.of` return the closest navigator above the widget. `m/showDialog` pushes its route on the root navigator by default (`.useRootNavigator true`), and `m/showModalBottomSheet` pushes on the closest one (`.useRootNavigator false`). Take the navigator inside the dialog or sheet builder, as in the example above, so `.pop` closes the dialog. With nested navigators (a go_router `ShellRoute`, for example), a navigator taken from the page that opened a dialog is a different one, and the Flutter `showDialog` docs close the dialog with the root navigator instead: `(.pop (m/Navigator.of ctx .rootNavigator true) result)`.
- Named route arguments are dynamic (untyped). Type-hint or validate after extraction to avoid dynamic warnings downstream.
- `go_router` extension methods like `.go` and `.push` require explicit qualification: `(-> ctx go/GoRouterHelper (.go "/path"))`.
