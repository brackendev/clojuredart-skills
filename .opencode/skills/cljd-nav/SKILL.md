---
name: cljd-nav
description: >-
  ClojureDart navigation patterns: named routes, go_router, Navigator API,
  tab navigation, and deep linking in Flutter.
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
    .onPressed (fn [] (.pushNamed navigator "/detail"))
    .child (m/Text "Go to Detail")))
```

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
    .onPressed (fn [] (go/GoRouter.of ctx .go "/detail/42"))
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
          .builder (f/build (detail/screen)))))
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
;; Show dialog
(f/widget
  :context ctx
  (m/ElevatedButton
    .onPressed
    (fn []
      (m/showDialog
        .context ctx
        .builder (f/build
                   (m/AlertDialog
                     .title (m/Text "Confirm")
                     .content (m/Text "Proceed?")
                     .actions
                     [(m/TextButton
                        .onPressed (fn [] (m/Navigator.of ctx .pop false))
                        .child (m/Text "Cancel"))
                      (m/TextButton
                        .onPressed (fn [] (m/Navigator.of ctx .pop true))
                        .child (m/Text "OK"))]))))
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
                     .child (m/Text "Bottom Sheet Content")))))
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
- `Navigator.of(context)` in ClojureDart is `(m/Navigator.of ctx)`. The `:get [m/Navigator]` directive is a cleaner alternative that avoids passing context manually.
- When using `:get [m/Navigator]` inside a dialog or bottom sheet, the navigator refers to the dialog's navigator scope, not the parent. Use `(m/Navigator.of ctx .rootNavigator true)` to reach the root navigator.
- Named route arguments are dynamic (untyped). Type-hint or validate after extraction to avoid dynamic warnings downstream.
- `go_router` extension methods like `.go` and `.push` require explicit qualification: `(-> ctx go/GoRouterHelper (.go "/path"))`.
