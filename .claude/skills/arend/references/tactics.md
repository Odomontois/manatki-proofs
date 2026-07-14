# Arend Standard Tactics / Metas Reference

Source: https://arend-lang.github.io/documentation/standard-tactics/ (crawled 2026-07-13)

This file documents the standard tactic/meta modules shipped with Arend's
standard library, one section per source page. Names, descriptions and code
examples are taken verbatim from the official docs; anything not shown on a
page (e.g. no example code) is noted as such rather than invented.

---

## 1. Meta (`\module Meta`)

Source: https://arend-lang.github.io/documentation/standard-tactics/meta

General-purpose, otherwise-unclassified tactics/metas: context manipulation
(`using`/`usingOnly`/`hiding`), case-splitting helpers (`cases`/`mcases`),
unfolding helpers (`unfold`/`unfold_let`), context search (`assumption`),
a `\let`-like sugar (`in`), and default class-field implementations
(`defaultImpl`).

### `using`
- **Description:** Adds `e₁, … eₙ` to the context before checking `e`.
- **Signature:** `using (e₁, … eₙ) e`
- **Example:** none shown in the docs.

### `usingOnly`
- **Description:** Replaces the context with `e₁, … eₙ` before checking `e`.
- **Signature:** `usingOnly (e₁, … eₙ) e`
- **Example:** none shown in the docs.

### `hiding`
- **Description:** Hides local bindings `e₁, … eₙ` from the context before checking `e`.
- **Signature:** `hiding (e₁, … eₙ) e`
- **Example:** none shown in the docs.

### `cases`
- **Description:** Works like `mcases`, but does not search for `\case` expressions or definitions in the expected type — it simply generates a case expression from the given arguments.
- **Signature:** `cases args default`, with parameters `addPath` and `name` for configuration.
- **Example:** none shown in the docs.

### `mcases`
- **Description:** Finds all invocations of definition `def` in the expected type and generates a `\case` expression that matches the arguments of those invocations.
- **Signature:** `mcases {def} args default \with { … }` — supports matching specific occurrences (e.g. `mcases {(def1,4),def2,(def3,1,2)}`) and an optional default clause.
- **Example:** none shown verbatim in the docs (only inline occurrence-selector patterns like `mcases {(def1,4),def2,(def3,1,2)}`).

### `unfold`
- **Description:** Unfolds functions `f₁, … fₙ` in the expected type before type-checking of `e`, and returns `e` itself.
- **Signature:** `unfold (f₁, … fₙ) e`
- **Example:** none shown in the docs.

### `unfold_let`
- **Description:** Unfolds `\let` expressions during type-checking.
- **Signature:** `unfold_let`
- **Example:** none shown in the docs.

### `assumption`
- **Description:** Searches for a proof in the context.
- **Signature:** `assumption {n}` or `assumption {n} a1 … ak` (variable/argument selection).
- **Example:** none shown in the docs.

### `in`
- **Description:** Syntactic sugar equivalent to `\let r => f x \in r` (commonly paired with tactics like `rewrite` or `simplify`).
- **Signature:** `f in x` or `(f₁, … fₙ) in x`
- **Example:** none shown in the docs.

### `defaultImpl`
- **Description:** Returns the default implementation of field `F` in class `C` applied to expression `E`.
- **Signature:** `defaultImpl C F E`
- **Example:** none shown in the docs.

---

## 2. Paths.Meta

Source: https://arend-lang.github.io/documentation/standard-tactics/paths-meta

Tactics for manipulating paths/equalities: rewriting the goal or a term with
an equality proof (`rewrite`/`rewriteF`/`rewriteI`/`rewriteEq`), proving
extensionality goals (`ext`), and simplifying goals involving `coe`/algebraic
identities (`simp_coe`/`simplify`).

### `rewrite`
- **Description:** Performs term rewriting using an equality proof, replacing occurrences of `a` in the expected type `T` with a variable `x`, obtaining `T[x/a]`, and returning `transportInv (\lam x => T[x/a]) p t`.
- **Signature:** `rewrite p t : T` where `p : a = b`
- **Example:**
```arend
\lemma +-assoc (a b c : Nat) : a + b + c = a + (b + c) \elim c
  | 0 => idp
  | suc c => rewrite (+-assoc a b c) idp

\lemma test1 (x y : Nat) (p : x = 0) (q : 0 = y) : x = y => rewrite p q
```

### `rewriteF`
- **Description:** Variant of `rewrite` that enforces use of the term's own type instead of the expected type.
- **Signature:** `rewriteF p t : T` where `p : a = b`
- **Example:**
```arend
\lemma test3 (x y : Nat) (p : 0 = x) (q : 0 = y) : x = y => rewriteF p q

\lemma test4 (x y : Nat) (p : 0 = x) (q : 0 = y) => rewriteF p q
```

### `rewriteI`
- **Description:** Applies `rewrite` with the inverted equality proof; equivalent to `rewrite (inv p) t`.
- **Signature:** `rewriteI p t`
- **Example:**
```arend
\lemma +-comm-rw (a b : Nat) : a + b = b + a
  | suc a, 0 => rewriteI (+-comm-rw a 0) idp

\lemma test2 (x y : Nat) (p : 0 = x) (q : 0 = y) : x = y => rewriteI p q
```

### `rewriteEq`
- **Description:** Advanced rewriting supporting algebraic equivalences (e.g. in monoids and categories). Like `rewrite`, it replaces literal occurrences of `a` in `T` with `b`, but it also finds and replaces occurrences up to algebraic equivalences.
- **Signature:** `rewriteEq p t : T` where `p : a = b`
- **Example:**
```arend
\lemma test1 {A : Monoid} (a b c d x : A) (B : A -> \Prop) (p : b * c = x) (t : B (a * x * d)) : B (((a * b) * (ide * c)) * (d * ide))
  => rewriteEq p t

\lemma test2 {C : Precat} {a b c d : C} (P : Hom a d -> \Prop) (f : Hom a b) (g : Hom b c) (g' : Hom c b) (h : Hom c d) (p : g ∘ g' = id c) (t : P ((h ∘ g) ∘ (id b ∘ f))) : P ((h ∘ id c ∘ g) ∘ (g' ∘ g ∘ f))
  => rewriteEq p (rewriteEq (idp {_} {(h ∘ g) ∘ (id b ∘ f)}) t)
```

### `ext`
- **Description:** Proves goals involving extensionality for functions, sigma-types, records, propositions, and types, transforming an extensionality goal into equivalent subgoal(s) appropriate to the goal's form.
- **Signature:** `ext` (takes an optional argument whose type depends on the shape of the goal); for records: `ext R { | f_1 => e_1 … | f_l => e_l }`
- **Example:** only the record syntax form is shown verbatim (`ext R { | f_1 => e_1 … | f_l => e_l }`); no full worked example given.

### `simp_coe`
- **Description:** Simplifies equality goals involving coercion operations (`coe`/`transport`), recursively simplifying goals with `coe` or `transport` in dependent function, sigma, and record types.
- **Signature:** `simp_coe` (expects one argument).
- **Example:** none shown in the docs.

### `simplify`
- **Description:** Simplifies the expected type (or, given an extra argument, a term's type) using algebraic identities.
- **Signature:** `simplify` (optional argument for additional context/term to simplify).
- **Example:**
```arend
\func test1 {M : Monoid} (x : M) : x = x * ide
  => simplify

\func test2 {M : Monoid} (x : M) : x * x = (x * (ide * x)) * ide
  => simplify

\func test3 {M : Monoid} (x : M) : ide * x * x * ide = x * ide * x * ide * ide
  => simplify

\func test9 {M : Monoid} (x y : M) (p : x * x = y) : x * (1 * x) = y * 1
  => simplify p
```

---

## 3. Function.Meta

Source: https://arend-lang.github.io/documentation/standard-tactics/function-meta

Small utilities for function application/iteration: implicit-argument-aware
analogues of Haskell's `$`, and a `repeat` combinator.

### `$`
- **Description:** Right-associative function application operator, analogous to Haskell's `$`, but (unlike a user-defined meta version) it also works correctly with implicit arguments. Partial application is possible via implicit lambdas, e.g. `__ $ a`.
- **Signature:** `\infixr 0 $ f a => f a` (shown as the "alternative"/naive definition one could write as a meta, which loses implicit-argument support — the built-in `$` retains it).
- **Example:** the naive-definition snippet shown is:
```
\meta \infixr 0 $ f a => f a
```

### `#`
- **Description:** Left-associative counterpart to `$`; same implicit-argument-aware function application, just left-associative.
- **Signature:** not detailed separately from `$` in the docs.
- **Example:** none shown in the docs.

### `repeat`
- **Description:** Applies a function repeatedly. Has two invocation modes: a counted mode, and a "keep going while it type-checks" mode.
- **Signature:**
  - `repeat {n} f x` — reduces to `x` if `n` is `0`, otherwise reduces to `repeat {x} f (f x)` if `n` is `suc x`.
  - `repeat f x` — attempts to typecheck `f x`; returns `x` on failure, or `f (repeat f x)` on success (this mode requires `f` to be a meta).
- **Example:** none shown in the docs.

---

## 4. Algebra.Meta

Source: https://arend-lang.github.io/documentation/standard-tactics/algebra-meta

Tactics for algebraic reasoning: proving chains of equalities (`equation`),
congruence closure (`cong`), and linear arithmetic over ordered rings (`linarith`).

### `equation`
- **Description:** Proves an equation `a_0 = a_{n+1}` using `a_1, … a_n` as intermediate steps.
- **Signature:** `equation a_1 … a_n`. Supports implicit arguments for step proofs, and can be combined with `using`, `usingOnly`, and `hiding` (single argument) to control the context. The first implicit argument can be a universe, or a subclass of `Monoid`, `AddMonoid`, or `Order.Lattice.Bounded.MeetSemilattice`.
- **Example:** none shown on the page.

### `cong`
- **Description:** Implements the congruence closure algorithm. For example, given assumptions `x = x'` and `y = y'`, it can prove `f x y = f x' y'`.
- **Signature:** implicit — draws on equality assumptions already present in the context.
- **Example:** none shown on the page.

### `linarith`
- **Description:** Solves systems of linear equations/inequalities in ordered rings using Fourier–Motzkin elimination.
- **Signature:** none specified beyond bare invocation.
- **Example:** none shown on the page.

---

## 5. Logic.Meta

Source: https://arend-lang.github.io/documentation/standard-tactics/logic-meta

Tactics/meta-predicates for logical reasoning: deriving contradictions
(`contradiction`), existential/universal quantifier sugar (`Exists`/`∃`,
`Given`, `Forall`/`∀`), and a context-driven `constructor`.

### `contradiction`
- **Description:** Attempts to derive a contradiction from the context when no arguments are provided. Supports implicit scope modifiers (`using`, `usingOnly`, `hiding`).
- **Signature:** `contradiction` or `contradiction {e}` where `e` is an explicit contradiction/witness.
- **Example:**
```arend
\lemma equality {x y : Nat} (p : x = y) (q : Not (x = y)) : Empty
  => contradiction

\lemma explicitProof (P : \Prop) (e : Empty) : P
  => contradiction {e}
```

### `Exists` (alias `∃`)
- **Description:** Represents existential quantification as `TruncP (\Sigma (x y z : _) B)`. Handles predicates, subtypes, and array membership.
- **Signature:** Implicit bindings `{x y z}`, explicit bindings `(x y z : A)`, or predicates/arrays, followed by the body.
- **Example:**
```arend
\lemma test1 : ∃ = (\Sigma) => idp

\lemma test3 : ∃ (x : Nat) (x = 0) = TruncP (\Sigma (x : Nat) (x = 0)) => idp

\lemma test7 (P : Nat -> \Type)
  : ∃ P = TruncP (\Sigma (x : Nat) (P x)) => idp
```

### `Given`
- **Description:** Similar syntax to `Exists`, but returns untruncated `\Sigma`-expressions instead of truncated (`TruncP`) ones.
- **Signature:** same argument forms as `Exists`.
- **Example:**
```arend
\func sigmaTest : Given (x : Nat) (x = 0) = (\Sigma (x : Nat) (x = 0)) => idp
```

### `Forall` (alias `∀`)
- **Description:** Returns `\Pi`-expressions for universal quantification, with syntax similar to `Exists`.
- **Signature:** Implicit bindings `{x y : Nat}`, explicit bindings `(x y : Nat)`, predicates, or arrays, followed by the body.
- **Example:**
```arend
\lemma piTest1 : ∀ (x y : Nat) (x = y) = (\Pi (x y : Nat) -> x = y) => idp

\lemma piTest2 : ∀ {x y : Nat} (x = y) = (\Pi {x y : Nat} -> x = y) => idp
```

### `constructor`
- **Description:** Returns tuples, `\new` expressions, or data type constructors depending on the expected type context.
- **Signature:** `constructor` followed by constructor arguments (values, or for data types, an implicit numeric index/selector plus values).
- **Example:**
```arend
\func tuple0 : \Sigma (\Sigma) (\Sigma) => constructor constructor constructor

\func tuple1 : \Sigma Nat Nat => constructor 0 1

\func data2 : D => constructor 0 1 2
  \where
    \data D | con {x : Nat} (y z : Nat)
```

---

## 6. Debug.Meta

Source: https://arend-lang.github.io/documentation/standard-tactics/debug-meta

Debugging helper metas: reading the clock (`time`), generating random
integers (`random`), and printing values during type-checking (`println`).

### `time`
- **Description:** Returns the current time in milliseconds, as an integer literal.
- **Signature:** `time`
- **Example:**
```
\func currentTime : Nat => time
\func currentTimeInt : Int => time
```

### `random`
- **Description:** Returns a random number as an integer literal. It can take arguments that specify the range of the returned random number.
- **Signature:** `random`, `random n`, or `random (l,u)`
- **Example:**
```
random -- returns a random number.

random n -- returns a random number between 0 and `n`

random (l,u) -- returns a random number between `l` and `u`
```

### `println`
- **Description:** Prints the argument to the console. In the IDE, it opens a tool window and displays the argument.
- **Signature:** `println arg`
- **Example:**
```
\import Debug.Meta
\import Meta (run)

\meta runTimed m => run {
  \let startTime => time,
  println m,
  println (time Nat.- startTime)
}

\func test => runTimed (114 Nat.+ 514)
```
