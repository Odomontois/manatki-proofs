# Arend: Concepts and Standard Library Primer

This is a conceptual reference for Arend, a homotopy-type-theory-flavored dependently
typed proof assistant developed by JetBrains. It assumes familiarity with Agda/Coq/Lean
and Martin-Löf type theory, and focuses on where Arend diverges from them. Code samples
are taken verbatim from the official tutorial and from the `arend-lib` standard library
source (the exact version this project's `arend.yaml` pins) unless marked "illustrative."

## 1. What makes Arend different from Agda/Coq/Lean

Arend's foundational choice is to be **homotopy type theory with a builtin interval
type**, with syntax "similar to cubical type theory." Concretely, this buys several
things Agda/Coq/Lean don't have out of the box:

- **Paths are functions out of an interval, not an inductively-defined `Eq`.** The
  identity type `a = a'` is defined as the type of functions `f : I -> A` with
  `f left == a` and `f right == a'` (definitionally). This is *not* Book-HoTT's path
  induction as a primitive; instead `J`, transport, congruence (`pmap`), and functional
  extensionality are all *derived*, and several of them get extra **definitional**
  (computational) equalities for free — e.g. `pmap id p == p` and
  `pmap (\lam x => g (f x)) p == pmap g (pmap f p)` hold by computation, not just
  propositionally. This is the single biggest practical difference from Agda/Coq/Lean,
  where such facts typically need an explicit proof (or don't hold at all without
  functional extensionality as an axiom).
- **Higher inductive types (HITs) are native**, with "path constructors" declared using
  the interval type directly inside ordinary `\data` (pattern-matching-with-conditions
  syntax), rather than being a separate/unsupported extension.
- **Univalence is a theorem, not an axiom**, thanks to a builtin `iso` primitive that
  turns an equivalence between `A` and `B` directly into a term of `A = B` with correct
  computational (`coe`/transport) behavior — no axiom-blocking of reduction the way
  postulated univalence does in other systems.
- **Two-dimensional universe hierarchy**: universes are indexed by both a predicative
  level *and* a homotopy (truncation) level, `\Type p h` a.k.a. `\h-Type p`. Because of
  this, if `a a' : A` and `A : \(h+1)-Type p`, then `a = a'` automatically lives in
  `\h-Type p` — you get "this equality is a mere proposition/set/..." for free from the
  ambient level, instead of having to prove and carry around separate h-level lemmas.
- **`\Prop` is impredicative**: propositions can quantify over types of any predicative
  level without bumping their own level.
- **Classes give Haskell-style automatic instance inference** (`\class`/`\instance`),
  layered on top of records (`\record`) which are literally named Sigma types with
  structural subtyping via "anonymous extensions" (`R { | f => e }`) — there is no
  separate notion of record *parameters* vs. *fields*.
- **No separate tactic/proof mode.** Unlike Coq, there is no mode switch between
  "term language" and "tactic language" — metas (Arend's analogue of tactics, e.g.
  `rewrite`) are ordinary term-level constructs that can be freely interleaved with
  regular expressions.
- **File-scoped, order-independent visibility.** A top-level definition's scope is the
  *entire* enclosing file (and transitively, files that import it), not just the code
  that follows it — closer to general-purpose languages than to Coq/Mizar-style
  sequential visibility. Combined with this, Arend permits circular dependencies
  *between definitions in the same library*, as long as actual recursive functions
  pass a structural termination check (à la Foetus/Abel's algorithm); circular
  dependencies *between libraries* are disallowed.
- **`\lemma` vs `\func`**: `\lemma` marks a definition whose *value* is irrelevant
  (proof irrelevance) and can be discarded after typechecking; it is only legal for
  definitions landing in `\Prop`. Otherwise `\lemma` and `\func` are syntactically
  identical — Arend has no separate `Theorem`/`Corollary` keywords the way Coq does;
  everything is `\func` or `\lemma`.

## 2. The interval type and path types

`I` is a builtin type in `Prelude` with two constructors `left, right : I`. It "looks
like" a two-element data type but is not one — pattern matching on `I` is forbidden
(it would let you prove `Empty = Unit`). Its only eliminator is `coe`, which
conceptually behaves like this (this is the tutorial's illustrative sketch of its
computation rule; `coe` itself is a language primitive):

```ard
\func coe (P : I -> \Type)
          (a : P left)
          (i : I) : P i \elim i
  | left => a
```

i.e. to build a dependent function out of `I`, it suffices to give its value at
`left` — `coe P a` automatically produces a term of `P i` for every `i`, and in
particular `coe P a right : P right`.

The identity/path type is built from `I` and `coe`. The type `a = {A} a'` is exactly
the type of functions `f : I -> A` such that `f left ==> a` and `f right ==> a'`
(`==>` = definitional/computational equality). Two operations move between paths and
such functions:

```ard
path (f : I -> A) : f left = f right   -- the (only) constructor of the '=' data type
@ (p : a = a') (i : I) : A             -- usually written infix: p @ i
```

They are mutually inverse **definitionally**:

```ard
path f @ i        ==> f i                 -- beta
path (\lam i => p @ i) ==> p              -- eta
```

`idp` (reflexivity) is just the constant function wrapped in `path`:

```ard
\func idp {A : \Type} {a : A} : a = a => path (\lam _ => a)
```

Because `path`/`@` are definitional inverses, everything built from them inherits
extra computation rules. This is the source of the "definitional functoriality" that
makes Arend's `pmap` behave better than a `transport`-based congruence:

```ard
\func pmap {A B : \Type} (f : A -> B) {a a' : A} (p : a = a') : f a = f a'
  => path (\lam i => f (p @ i))

-- these hold by idp, i.e. definitionally, not just propositionally:
\func pmap-idp {A : \Type} {a a' : A} (p : a = a') : pmap {A} (\lam x => x) p = p => idp
```

`transport` is defined via `coe` applied to the path viewed as a family over `I`:

```ard
\func transport {A : \Type} (B : A -> \Type) {a a' : A} (p : a = a') (b : B a) : B a'
  => coe (\lam i => B (p @ i)) b right
```

Function extensionality is a one-liner, again because a path is literally a function
out of `I`:

```ard
\func funExt {A : \Type} (B : A -> \Type) (f g : \Pi (x : A) -> B x)
             (h : \Pi (x : A) -> f x = g x) : f = g
  => path (\lam i a => h a @ i)
```

`I` itself is contractible: `left = right` is provable using `coe` (not as an axiom):

```ard
\func left=i (i : I) : left = i => coe (\lam i => left = i) idp i
\func left=right : left = right => left=i right
```

`n`-dimensional cubes are just `\Sigma I ... I` (`n`-fold product of `I`); they show
up as the "shape" of path-constructor arguments in higher inductive types (section 4).

## 3. Univalence: `iso`, `Equiv`/`QEquiv`, and equivalence-to-equality

Arend represents "equivalence between types" with a small hierarchy of records in
`arend-lib`'s `Equiv` module (`src/Equiv.ard`):

```ard
\record Map {A B : \Type} (\coerce f : A -> B)

\record Section (ret : B -> A) (ret_f : \Pi (x : A) -> ret (f x) = x) \extends Map

\record Retraction (sec : B -> A) (f_sec : \Pi (y : B) -> f (sec y) = y) \extends Map

\record Equiv \extends Section, Retraction { ... }   -- coherent adjoint equivalence

\record QEquiv \extends Equiv { | sec => ret }        -- quasi-equivalence: one map
                                                       -- 'ret' serves as both sides'
                                                       -- inverse (sec is set equal to ret)
```

`QEquiv` is usually the one you construct by hand, via `\cowith`, since it only asks
for `f`, `ret`, `ret_f`, `f_sec`:

```ard
-- from this project's src/circle.ard
\func plus-one-quequiv : QEquiv {Int} {Int} \cowith
  | f x => x + 1
  | ret x => x - 1
  | ret_f _ => equation.abGroup
  | f_sec _ => equation.abGroup
```

The mechanism that turns such an equivalence into an actual path `A = B` is a builtin
called `iso`. Given `f : A -> B`, `g : B -> A`, and proofs that they are mutually
inverse, `iso f g ret_f f_sec` is a term of `I -> \Type` with the computation rules
`iso f g p q left ==> A` and `iso f g p q right ==> B`. Wrapping it in `path` therefore
directly produces `A = B` — this *is* Arend's univalence, built into path types the
same way Cubical Type Theory's `Glue`/`ua` is, rather than being postulated:

```ard
-- src/Equiv/Univalence.ard in arend-lib
\func QEquiv-to-= {A B : \Type} (e : QEquiv {A} {B}) : A = B
  => path (iso e.f e.ret e.ret_f e.f_sec)

\func =-to-QEquiv {A B : \Type} (p : A = B) : QEquiv {A} {B} \cowith
  | f    => transport (\lam X => X) p
  | ret  => transport (\lam X => X) (inv p)
  | ret_f a => inv (transport_*> _ p (inv p) a) *> pmap (transport _ __ a) (*>_inv p)
  | f_sec b => inv (transport_*> _ (inv p) p b) *> pmap (transport _ __ b) (inv_*> p)

\func Equiv-to-= {A B : \Type} (e : Equiv {A} {B}) : A = B => QEquiv-to-= (QEquiv.fromEquiv e)

-- univalence itself: paths between types ARE (equivalent to) equivalences
\func univalence {X Y : \Type} : QEquiv {X = Y} {Equiv {X} {Y}} => ...
```

A canonical use is defining a type family over a HIT so that going around a loop
performs a nontrivial automorphism — e.g. the standard proof that `pi1(Circle) = Int`
computes the universal cover as:

```ard
\func code (x : Sphere1) : \Set0
  | base1 => Int
  | loop i => iso isuc ipred ipred_isuc isuc_ipred i
```

i.e. transporting `0 : Int` around `loop` computes to `1`, definitionally — you can
literally run this as a program (`coe (shift-circle Loop) 0 right ==> 1`), which is
exactly what `src/circle.ard` in this project does with `test-calc`/`println`.

Note: `iso`/`path` give you `A = B` in `\Type`; for `\Set`/`\Prop`-level types the same
`iso` mechanism underlies `SigmaExt`-style extensionality lemmas throughout the library.

## 4. Higher inductive types (HITs)

A HIT is a `\data` definition whose constructors can include **path constructors**:
constructors indexed by `I` (or several `I`s, for higher cells), with `\with`/`\elim`
clauses pinning down what they compute to at `left`/`right` (the "boundary"). This
is ordinary pattern-matching syntax — no separate HIT sub-language.

The circle:

```ard
\data Circle
  | base
  | loop I \with {
    | left => base
    | right => base
  }
```

`loop` behaves like an ordinary function of its `I` argument: `loop left` and
`loop right` compute (reduce) to `base`. To get the actual path `base = base`, wrap it:
`\func simpleLoop : base = base => path loop`.

A 2-dimensional cell needs two `I` arguments and four boundary conditions (since the
boundary of a square has four sides):

```ard
\data S2
  | base2
  | loop2 I I \with {
    | left, _ => base2
    | right, _ => base2
    | _, left => base2
    | _, right => base2
  }
```

Higher spheres are more commonly built via **suspension**:

```ard
\data Susp (A : \Type)
  | south
  | north
  | merid A (i : I) \elim i {
    | left  => north
    | right => south
  }

\func Sphere (n : Nat) : \Type \lp \oo
  | 0 => Susp Empty
  | suc n => Susp (Sphere n)
```

The torus, by identifying opposite sides of a square directly:

```ard
\data Torus
  | point
  | line1 I \with { left => point | right => point }
  | line2 I \with { left => point | right => point }
  | face I I \with {
    | left, i => line2 i
    | right, i => line2 i
    | i, left => line1 i
    | i, right => line1 i
  }
```

Functions out of a HIT are defined by pattern matching as usual, but the clauses must
respect the equations imposed by the path constructors (the typechecker verifies this
and errors if not):

```ard
\func circRec {B : \Type} {b : B} (l : b = b) (x : Circle) : B \elim x
  | base => b
  | loop i => l @ i
```

The corresponding **dependent** version (higher induction) needs a "dependent loop" —
either a heterogeneous `Path (\lam i => B (loop i)) b b`, or equivalently (and often
more convenient) a homogeneous path `transport B (path loop) b = b`:

```ard
\func circInd (B : Circle -> \Type) (b : B base)
              (l : transport B (path loop) b = b) (x : Circle) : B x \elim x
  | base => b
  | loop i => (concat {\lam i => B (loop i)}
                      (path (\lam i => coe (\lam j => B (loop j)) b i)) l) @ i
```

**Truncated HITs**: `\truncated \data ... : \Set` (or `\Prop`, `\n-Type`, ...) declares
a HIT that is automatically truncated to the given homotopy level, which lets you
eliminate into that level while only handling constructors up to dimension `level+1`.
The canonical example is the set-quotient:

```ard
\truncated \data Quotient {A : \Type} (R : A -> A -> \Type) : \Set
  | in~ A
  | ~-equiv (x y : A) (R x y) (i : I) \elim i {
    | left  => in~ x
    | right => in~ y
  }
```

`arend-lib`'s own HITs live under the `Homotopy` namespace (spheres in
`Homotopy.Sphere`, pushouts in `Homotopy.Pushout`, the Eilenberg–MacLane space `K1` in
`Homotopy.K1`, etc. — see the module list in section 6).

## 5. Level polymorphism and universes

Arend's universe hierarchy has **two independent numeric parameters**:

- a **predicative level** `p` (like Agda/Coq's universe levels — tracks size/impredicativity),
- a **homotopy (truncation) level** `h` (tracks how far up the ∞-groupoid structure a
  type is truncated: -1 = proposition, 0 = set, 1 = groupoid, ..., `\oo` = untruncated).

The universe of level `(p, h)` is written `\h-Type p`, equivalently `\Type p h`.
Special-cased notations:

- `\Set p` = `\0-Type p` (sets — types truncated to homotopy level 0)
- `\Prop` = the smallest universe, homotopy level -1, and **impredicative** (no
  predicative index at all — a `\Prop` can quantify over types of any predicative
  level without raising its own level)
- `\Type` with no explicit indices lets the typechecker infer the minimal `(p, h)`
- `\oo` is the top (unbounded/no-truncation) homotopy level

Level expressions are built from numerals and the implicit per-definition level
parameters `\lp` (predicative) and `\lh` (homotopy) using two operations:
`\suc l` and `\max l1 l2`. Every definition implicitly carries both `\lp` and `\lh`
as level parameters, the same way Agda has an implicit `ℓ`. Examples straight from
the tutorial:

```ard
\func tt : \Type2 => \Type0 -> \Type1

-- 'id' is implicitly level-polymorphic in the level of A:
\func id (A : \Type) (a : A) => a
-- equivalent, spelling out the implicit level parameter:
\func id (A : \Type \lp) (a : A) => a

\func type : \Type => \Type
-- typechecker infers: \func type : \Type (\suc \lp) => \Type \lp

\func test0 : \Type (\max (\suc (\suc \lp)) 4) => \Type (\max \lp 3) -> \Type (\suc \lp)
```

You can pin a specific level at a call site either positionally (if it isn't a
numeral, pass it as the first explicit argument) or with `\levels p h`:

```ard
\func test5  => id (\suc \lp) (\Type \lp) Nat
\func test5' => id (\levels (\suc \lp) _) (\Type \lp) Nat
\func test6  => id (\levels 2 _) \Type1 \Type0
```

The homotopy-level side of the hierarchy is what gives Arend "free" h-level tracking:
if `A : \(h+1)-Type p`, then for `a a' : A` the path type `a = a'` automatically lives
in `\h-Type p` — no separate `isSet`/`isProp` proof obligation is needed to know an
equality type's h-level; it falls out of the ambient universe. Universes are also
cumulative: a type in `\h-Type p` is also in `\h'-Type p'` for any `h' >= h`, `p' >= p`.

`\data`/`\class`/`\record` levels are computed from their *non-implemented* fields and
constructor parameters — implementing a field can shrink the resulting level:

```ard
\class Magma (A : \Type)
  | \infixl 6 ** : A -> A -> A
-- Magma \lp        : \Type (\suc \lp)   -- A is still abstract
-- Magma \lp Nat    : \Type0             -- A implemented as Nat, no universe left inside
```

## 6. Projects and libraries: `arend.yaml`, `\import`

An **Arend library** is a directory containing a mandatory `arend.yaml` manifest plus
a source tree. Standard layout:

- root: `arend.yaml` (manifest)
- `src/` (configurable via `sourcesDir`): `.ard` source files
- `.bin`/`bin/` (`binariesDir`): compiled `.arc` typechecker output
- `ext/` (`extensionsDir`, optional): Java classes implementing metas (Arend's
  tactic-equivalent), via `org.arend.ext.ArendExtension`
- `test/` (`testsDir`, optional): tests for library-defined metas

`arend.yaml` fields:

```yaml
langVersion: VERSION          # e.g. ">= 1.9", or a range ">= V1, <= V2"
version: VERSION              # this library's own version
sourcesDir: PATH
binariesDir: PATH
extensionsDir: PATH           # optional
testsDir: PATH                # optional
extensionMainClass: CLASS_PATH  # optional, Java class implementing ArendExtension
dependencies: [NAME_1, ..., NAME_k]
```

This project's own `arend.yaml`:

```yaml
sourcesDir: src
binariesDir: .bin
langVersion: 1.11
dependencies: [arend-lib]
```

**Circular dependencies between two different libraries are disallowed** (unlike
circular dependencies between definitions inside one library, which are permitted as
long as recursive calls pass Arend's structural termination check). To typecheck
multiple libraries from the console (outside IntelliJ), put them in one directory and
point the `-L` flag at it; inside IntelliJ, the Arend plugin's "Arend" module type
keeps `arend.yaml` and IDE module settings in sync automatically.

### `\import` / `\open` syntax

`\import X` makes the definitions of file/module `X` visible in the current file, and
(unless suppressed) also **opens** them into scope — everything `\open` can do,
`\import` can do too:

```ard
-- sourcesDir contains Test.ard (defining foobar, foobar2)
-- and a TestDir/ directory with Test.ard and Test2.ard inside it.

\import Test (foobar \as foobar', foobar2)  -- import + open only these two, renaming one
\import TestDir.Test                        -- module path = directory path with dots
\import TestDir.Test2()                     -- import (make visible / typecheck-visible)
                                             -- but do NOT open any names
```

`\open` (also usable standalone, not just via `\import`) supports selecting, renaming,
and hiding names:

```ard
\open M1                       -- open everything from module M1
\open M1(f, g)                 -- open only f and g
\open M2 \hiding (t')          -- open everything except t'
\open M3 (t \as M3_t)          -- open only t, renamed to M3_t
\open M4 \using (t \as M4_t)   -- open everything, renaming t to M4_t
```

Names can always be reached with an explicit qualified prefix even when not opened
(`M1.h`), and every definition implicitly defines its own nested module via `\where`,
so `f.g` reaches a helper `g` defined in `f`'s `\where` block. `\module M \where { ... }`
groups definitions under an explicit namespace the same way a file does.

**Practical note for this project**: `arend-lib`'s h-level utilities
(`isProp`, `ofHLevel_-1+`, etc.) live in the top-level module `HLevel`
(`\import HLevel`), *not* under a `Homotopy` namespace — there is no `Homotopy/HLevel.ard`
in the pinned `arend-lib` checkout used here (only `Homotopy/` HIT-space modules like
`Homotopy.Sphere`, `Homotopy.Pushout`, etc., plus a separate top-level `HLevel.ard`).
An import of `Homotopy.HLevel` will not resolve against this library version; use
`\import HLevel` instead. Likewise, univalence-related helpers (`QEquiv-to-=`,
`Equiv-to-=`, `univalence`) live in `Equiv.Univalence`, imported as
`\import Equiv.Univalence`, separately from the base `Equiv` module.

Metas worth knowing about early: `Meta`, `Paths.Meta`, `Function.Meta`,
`Algebra.Meta`, `Logic.Meta` provide surface-syntax shortcuts (e.g. `rewrite`) that
expand into ordinary kernel terms; import them alongside the module they augment.

## 7. arend-lib top-level modules

Enumerated from the `src/` directory of the exact `arend-lib` checkout this project
depends on (`/Users/odomontois/Prog/projects/arend/arend-lib`, matching
`JetBrains/arend-lib`). Descriptions marked **[doc]** are confirmed by the official
tutorial/features pages; the rest are **[inferred]** from directory/file names and
brief header-comment/import inspection only (no dedicated prose documentation found
for them).

| Module | Description |
|---|---|
| `Paths` | **[doc]** Core path/equality combinators: `idpe`, `pmap`, `pmap2`, `transport`, `transportInv`, `transport2`, `inv`, `*>` (concatenation), etc. — the basic algebra of the `=` type. |
| `Equiv` | **[doc]** Equivalence records: `Map`, `Section`, `Retraction`, `Equiv`, `QEquiv`, `Embedding`, `Surjection`, `ESEquiv`. |
| `Equiv.Fiber` | **[inferred]** Homotopy fibers of maps (used to characterize contractibility of `Section`/`Retraction`). |
| `Equiv.HalfAdjoint` | **[inferred]** Half-adjoint equivalences (the "coherent" equivalence structure classically used to prove `isProp(isEquiv f)`). |
| `Equiv.Path` | **[inferred]** Equivalences/paths between path types themselves (path algebra lemmas used by `Equiv`/`HLevel`). |
| `Equiv.Sigma` | **[inferred]** Equivalences involving Sigma types (e.g. reassociation, fiberwise equivalence). |
| `Equiv.Univalence` | **[doc]** Univalence: `=-to-Equiv`, `=-to-QEquiv`, `QEquiv-to-=`, `Equiv-to-=`, `univalence`. |
| `HLevel` | **[doc]** Homotopy-level (h-level) predicates: `isProp`, `ofHLevel_-1+`, and related lemmas (`isProp=>isSet`, etc.). |
| `Function` | **[doc]** Basic function combinators: `id`, composition (`o`, `-o`, `o-`), `iterl`/`iterr`. |
| `Logic` | **[doc]** Core propositional logic on top of `\Prop`: `Empty`, `Not`, `Either`/`Or`, existentials, `Dec`, decidability. |
| `Logic.Classical` | **[inferred]** Classical-logic principles (excluded middle style reasoning), for contexts that assume them. |
| `Logic.FirstOrder` | **[inferred]** First-order logic encodings/utilities. |
| `Logic.Rewriting` | **[inferred]** Rewriting-system-style support for logic (term rewriting relations, possibly backing the `rewrite` meta). |
| `Logic.PropFin` | **[inferred, single file]** Finiteness for propositions. |
| `Set` | **[doc]** Decidability/set-theoretic base: `Dec`, and the set-level ("h-level 0") API. |
| `Set.Category` | **[inferred]** The category of sets. |
| `Set.Countable` | **[inferred]** Countable sets. |
| `Set.Fin` | **[inferred]** Finite sets (`FinSet`, `DecFin`, `SigmaFin` per `Set.ard`'s import). |
| `Set.Filter` | **[inferred]** Filters on sets (used for topology/analysis limit notions). |
| `Set.Hedberg` | **[inferred]** Hedberg's theorem (decidable equality ⇒ set). |
| `Set.Partial` | **[inferred]** Partial sets / partial elements. |
| `Set.Subset` | **[inferred]** Subsets, subset relations. |
| `Relation.Equivalence` | **[inferred]** Equivalence relations. |
| `Relation.Truncated` | **[inferred]** Propositionally-truncated relations. |
| `Order` | **[inferred, dir only]** Order theory: `PartialOrder`, `LinearOrder`, `StrictOrder`, `Lattice`, `BooleanAlgebra`, `HeytingAlgebra`, `Directed`, `Biordered`, `Lexicographical`, `Category` (order as category). |
| `Algebra` | **[inferred, dir + confirmed via `Algebra/Algebra.ard`]** Algebraic structures: `Monoid`, `Group`, `Ring`, `Semiring`, `Field`, `Domain`, `Module`, `Pointed`, `Linear` (linear algebra), plus an `AAlgebra` (associative algebra) class in `Algebra.ard` itself, `MulOrdered`/`Ordered` variants, and `FinSuppFunc` (finitely-supported functions). |
| `Category` | **[doc, header confirmed]** Category theory core: `\class Precat` (`Ob : \hType`, `Hom`, ...); submodules add `Functor`, `Adjoint`, `Limit`, `KanExtension`, `Yoneda`, `CartesianClosed`, `Topos` (+ `Category.Topos` subdir), `Comma`, `Slice`, `Product`, `Subcat`, `Subobj`/`SubobjectPoset`, `Displayed`, `PreAdditive`, `Simplex`, `Factorization`, `Coreflection`, `Solver` (category-theoretic proof automation meta). |
| `Homotopy` | **[doc, via tutorial Part II]** Spaces and homotopy theory: HIT building blocks (`Pointed`, `Space`, `Fibration`, `Square`, `Cube`), core HIT constructions (`Pushout`, `Suspension`, `Join`, `Image`), spheres/circle (`Sphere`, `Sphere.Circle`), the fundamental-group / Eilenberg–MacLane machinery (`Loop`, `Truncation`, `Connected`, `K1`, `EckmannHilton`), `Hopf` fibration, `Torus`, and localization theory under `Homotopy.Localization` (`Modality`, `Accessible`, `BlakersMassey`, `Connected`, `Equiv`, `Separated`, `Universe`). |
| `Data` | **[inferred]** Concrete data structures: `Array`, `Bool`, `Fin`, `List`, `Maybe`, `Or`, `Sigma`, `SubList`, `Shifts`, `SeqColimit` (sequential colimits). |
| `Arith` | **[inferred]** Concrete number systems and their arithmetic: `Nat`, `Int`, `Rat`, `Real` (+ `Arith.Real` subdir), `Complex`, `Bool`, `Fin`, `Prime`. |
| `Combinatorics` | **[inferred]** `Binom` (binomial coefficients), `Factorial`. |
| `Analysis` | **[inferred]** Real/functional analysis: `Limit`, `FuncLimit`, `Derivative`, `StrongDerivative`, `PowerSeries`, `Series`, plus `Analysis.Calculus` and `Analysis.Measure` subdirs. |
| `Topology` | **[inferred]** Point-set/metric topology: `TopSpace`, `MetricSpace`, `UniformSpace`, `CoverSpace`, `Locale`, `Compact`, `Lipschitz`, `RatherBelow`, and normed-structure variants `NormedAbGroup`, `NormedAlgebra`, `NormedModule`, `NormedRing`, `TopAbGroup`, `TopModule`, `TopRing`, plus `BanachSpace`, `ContGerm` (continuous germs), `Partial`. |
| `AG` | **[inferred]** Algebraic geometry: `Projective` (projective varieties/space), `Scheme`. |
| `Debug` | **[inferred, header seen]** Developer utilities, e.g. `runTimed` (timed evaluation via the `Debug.Meta`/`Meta` extensions). |
| `Operations` | **[inferred, header seen]** Generic notation classes for common operator overloading, e.g. `HasProduct` (`⨯`), `HasCoproduct` (`⨿`), with instances for `\Type`. |

Notes on discovering this list: the rendered `arend-lang.github.io/arend-lib` landing
page itself is just a version-picker (linking to generated Scaladoc-style HTML class
indexes and a class graph, not prose), so it does not enumerate modules with
descriptions. The list above was built by cloning `JetBrains/arend-lib` and walking
`src/`, then reading the header/import lines of one representative file per top-level
module to confirm what each actually contains; only `Paths`, `Equiv`,
`Equiv.Univalence`, `HLevel`, `Function`, `Logic`, `Set`, `Category` (partially), and
`Homotopy` (via the tutorial's fundamental-group-of-the-circle walkthrough) have
prose-level documentation backing their description above.

## Sources

- Tutorial Part I (Basics, Equality, Universes) — fetched via raw source at
  `arend-lang/site` (`src/documentation/tutorial/PartI/*.md`); the rendered
  `arend-lang.github.io/documentation/tutorial/PartI` page itself only exposed nav
  chrome to automated fetching, so the Jekyll source markdown was used instead.
- Tutorial Part II (Spaces/HITs, Propositions and Sets) — same approach,
  `src/documentation/tutorial/PartII/*.md`.
- Getting Started: Arend Features — `src/documentation/getting-started/arend-features.md`.
- Getting Started: Libraries — `src/documentation/getting-started/libraries.md`.
- `arend-lib` module structure — cloned from `https://github.com/JetBrains/arend-lib`
  (and cross-checked against the version already vendored locally at
  `/Users/odomontois/Prog/projects/arend/arend-lib`, which is the exact version this
  project's `arend.yaml` depends on).