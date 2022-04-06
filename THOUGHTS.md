# Problem

`Namespace#reference` contains "last-Var-wins" logic that doesn't account for namespaces with Vars populated via AOT compilation.
The result is that when a compiled name clashes with a clojure.core name, then the mapping from the symbol to the Var is replaced 
with the core Var.

There are two manifestations of this problem depending on certain AOT conditions:

1) The namespace `fogus.oddity` containing a clashing function _definition only_ `abs`
2) The namespace `fogus.oddity` containing the clashing function `abs` definition and internal calls to it

## Background

Related ticket https://clojure.atlassian.net/browse/CLJ-1241 seems to have originally encountered this problem but the fix was not a fix.

https://ask.clojure.org/index.php/11672/fastmath-errors-in-1-11

### **Manifestatiion #1 - Definition Only Case**

#### REQUIRE Phase

- code `(require 'fogus.oddity)`
- `require` calls `load-lib` which initiates AOT code load `fogus.oddity__init` which initializes `fogus.oddity$abs` function class
- AOT `#'fogus.oddity/abs` created and interned with `RT.var`
- AOT `#'fogus.oddity/abs` has its root bound to new `fogus.oddity$abs` function instance
- `fogus.oddity__init` calls `refer` with every sym->var mapping in clojure.core against `fogus.oddity` namespace
- `Namespace.reference` resets the mapping for `abs` in fogus.oddity's `mappings` from `#'fogus.oddity/abs` to `#'clojure.core/abs`

#### Symbol->Var Resolution Phase

- code `fogus.oddity/abs`
- Symbol resolution in `Compiler/resolveIn` fails because it calls `Namespace.findInternedVar` which compares `fogus.oddity` name and `#'clojure.core/abs` `ns` field and finds a mismatch
- Compiler reports `No such var: fogus.oddity/abs`




### **Manifestatiion #2 - Definition and Call Case (e.g. `abs` called in `-main`)**

#### REQUIRE Phase

- code `(require 'fogus.oddity)`
- `require` calls `load-lib` which initiates AOT code load `fogus.oddity__init` which initializes `fogus.oddity$abs` function class
- AOT `#'fogus.oddity/abs` created and interned with `RT.var`
- AOT `#'fogus.oddity/abs` has its root bound to new `fogus.oddity$abs` function instance
- `fogus.oddity__init` calls `refer` with every sym->var mapping in clojure.core against `fogus.oddity` namespace
- `Namespace.reference` resets the mapping for `abs` in fogus.oddity's `mappings` from `#'fogus.oddity/abs` to `#'clojure.core/abs`
- `fogus.oddity__init` initiates AOT code load for `fogus.oddity$_main` function class
- `Namespace.intern` called again in `fogus.oddity$_main` function class static segment
- `Namespace.intern` looks up var in `fogus.oddity` namespace and gets `#'clojure.core/abs`
- Because var nses do not match, a new `#'fogus.oddity/abs` is created in `fogus.oddity` namespace with an unbound root
- `Namespace.intern` reports `WARNING: abs already refers to: #'clojure.core/abs in namespace: fogus.oddity, being replaced by: #'fogus.oddity/abs`

#### Symbol->Var Resolution Phase

- code `fogus.oddity/abs`
- Symbol resolution in `Compiler/resolveIn` succeeds because it calls `Namespace.findInternedVar` which compares `fogus.oddity` name and `#'fogus.oddity/abs` `ns` field and finds a match
- Symbol resolution returns `#'fogus.oddity/abs` with unbound root

#### Call Phase

- code `(fogus.oddity/abs "a")`
- Evaluation results in `Attempting to call unbound fn: #'fogus.oddity/abs` error

