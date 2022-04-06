# Problem

`Namespace#reference` contains "last-Var-wins" logic that doesn't account for namespaces with Vars populated via AOT compilation.
The result is that when a compiled name clashes with a clojure.core name, then the mapping from the symbol to the Var is replaced 
with the core Var.

There are two manifestations of this problem depending on certain AOT conditions:

1) The namespace `fogus.oddity` containing the clashing function `abs` is compiled with or without `:genclass`
2) The namespace `fogus.oddity` containing the clashing function `abs` is compiled with `:genclass` and also contains a `-main` fn

The call to the `-main` function need never occur.

# non -main case

LOAD Phase
----------

- `#'fogus.oddity/abs` created
- `#'fogus.oddity/abs` interned
- `#'fogus.oddity/abs` has its root to new `fogus.oddity$abs` instance

REQUIRE Phase
-------------

- code `(require 'fogus.oddity)`
- `Namespace.reference` resets the mapping for `abs` in fogus.oddity's `mappings` from `#'fogus.oddity/abs` to `#'clojure.core/abs`

Symbol->Var Resolution Phase
----------------------------

- code `fogus.oddity/abs`
- Sybol resolution in `Compiler/resolveIn` fails because it calls `Namespace.findInternedVar` which compares ns name and var's ns field and finds a mismatch
- Compiler reports `No such var: fogus.oddity/abs`

# -main case

LOAD Phase
----------

- `#'fogus.oddity/abs` created
- `#'fogus.oddity/abs` interned
- `#'fogus.oddity/abs` has its root to new `fogus.oddity$abs` instance

REQUIRE Phase
-------------

- code `(require 'fogus.oddity)`
- `Namespace.reference` resets the mapping for `abs` in fogus.oddity's `mappings` from `#'fogus.oddity/abs` to `#'clojure.core/abs`
- `Namespace.intern` called again
- `Namespace.intern` looks up var in `fogus.oddity` namespace and gets `#'clojure.core/abs`
- Because var nses do not match, a new `#'fogus.oddity/abs` is created in `fogus.oddity` namespace with an unbound root
- `Namespace.intern` reports `WARNING: abs already refers to: #'clojure.core/abs in namespace: fogus.oddity, being replaced by: #'fogus.oddity/abs`

Symbol->Var Resolution Phase
----------------------------

- code `fogus.oddity/abs`
- Symbol resolution returns `#'fogus.oddity/abs` with unbound root

Call Phase
----------

- code `(fogus.oddity/abs "a")`
- Evaluation results in `Attempting to call unbound fn: #'fogus.oddity/abs` error

