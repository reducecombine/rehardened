# Rehardened

This repo aims to contain useful snippets and advice for never having to restart a JVM again.

It requires quite some head scratching to make the Reloaded workflow avoid a variety of pitfalls. I've observed that Clojure programmers keep encountering these, so I decided to share some knowledge in this repo.

PRs with further advice are absolutely welcome!

## General advice

### Use an external library which defines your refresh/reset functions:

* https://github.com/stuartsierra/component.repl
  * Or https://github.com/weavejester/reloaded.repl for supporting [suspendable](https://github.com/weavejester/suspendable)
* https://github.com/weavejester/integrant-repl

A very important point of those is that they are outside of your source-paths/refresh-paths, so they don't ever get unloaded.

### Don't create any helper function for refreshing

At least not in your source-paths/refresh-paths; author instead a library similar to the ones mentioned above.

### Use a reset/reload snippet that requires zero context or state (loaded code, current ns, etc)

Normally your IDE can be assigned a keyboard shortcut to a custom reload command. Don't bind `(reset)` to that shortcut, since it depends on the `user` ns to be loaded. Something like `(com.stuartsierra.component.repl/reset)` needs less context. 

### Catch errors

* Switching between branches can confuse c.t.n.r.refresh. You should catch that FileNotFoundException and issue a c.t.n.r.clear.

* Sierra's Component doesn't perform any cleanup of unsuccessfully started systems. Catch any errors and attempt a cleanup of your half-started system.
  * Else you'll see things like HTTP servers encountering a port-already-taken situation.

### Distinguish between refresh and reset

It's tempting to simplify things and just always `reset`. But it's useful to also `refresh`, since it's vastly faster and sufficient for developing code and running tests.
  * I use Command+Option+Enter for `refresh`, Control+Enter for `reset`.

### Avoid stateful or side-effectful namespaces to the extreme

If you are familiar with Reloaded, probably you already avoid side-effectful namespaces:

* No top-level state: `(def x (atom {}))`
* No top-level side effects: `(register! a-thing)` 

You can take this philosophy to the extreme and isolate:

* The `clojure.core/extend*` family of functions
* `defrecord`s (and similar) containing protocol implementations
  * Make your `defrecord`s anemic, extend them later
* `defmethod`
* Avoid side-effectful imports: e.g. a `:require` of a namespace only containing `defmethod`s

Launch all of those in your Component `start` method. Similarly, undo all state in your `stop` method, e.g. undefine the multimethod implementations.

It's also worth looking into metadata-based protocol extension, for both defrecords, and plain maps.

In particular, plain maps instead of `defrecord`s are drastically less likely to have code loading issues: classes are not guaranteed to be garbage-collected, while objects are.

### Don't refresh-on-save

Having some setup for issuing a `refresh` on every save (Command+S) is generally a recipe for disaster. I recommend instead that you create a composite command in your editor that intentfully saves + refreshes.

In my case, I hit:

* Command+S for just saving
* Command+Option+Enter for saving _and_ refreshing

That way, I can keep the distinction while still needing few keystrokes.

## Snippets

The following are battle-hardened snippets that you can bind to your IDE. I know they look kind of funny but there's a reason for every aspect you can see.

#### Refresh

```clojure
(clojure.core/when-let [v (do
                            (clojure.core/require 'clojure.tools.namespace.repl)
                            (clojure.core/require 'com.stuartsierra.component.repl)
                            ((clojure.core/resolve 'clojure.tools.namespace.repl/set-refresh-dirs) "src" "test")
                            ((clojure.core/resolve 'clojure.tools.namespace.repl/refresh)))]
  (clojure.core/when (clojure.core/instance? java.lang.Throwable v)
    (clojure.core/when (clojure.core/instance? java.io.FileNotFoundException v)
      ((clojure.core/resolve 'clojure.tools.namespace.repl/clear)))
    (throw v)))
```

#### Reset

```clojure
(do
  (clojure.core/require 'clojure.tools.namespace.repl)
  (clojure.core/require 'com.stuartsierra.component.repl)
  ((clojure.core/resolve 'clojure.tools.namespace.repl/set-refresh-dirs) "src" "test")
  (try
    ((clojure.core/resolve 'com.stuartsierra.component.repl/reset))
    (catch java.lang.Throwable v
      (clojure.core/when (clojure.core/instance? java.io.FileNotFoundException v)
        ((clojure.core/resolve 'clojure.tools.namespace.repl/clear)))
      (clojure.core/when ((clojure.core/resolve 'com.stuartsierra.component/ex-component?) v)
        (clojure.core/let [stop (clojure.core/resolve 'com.stuartsierra.component/stop)]
          (clojure.core/some-> v clojure.core/ex-data :system stop)))
      (throw v))))
```

Note that these assume Component, and a `["src" "test"]` source path structure.

Alternatively to the "IDE snippets" approach, you can bundle code like this in your own library, similarly to the libraries mentioned in the first section. Personally I don't because I don't need it - it's more agile to have a snippet that I can continuously refine without releasing .jars, etc.
