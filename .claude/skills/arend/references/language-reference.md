# Arend Language Reference

Condensed, example-driven reference distilled from the official Arend documentation's
Language Reference section (https://arend-lang.github.io/documentation/language-reference/).
This is ground truth for an LLM writing/reading `.ard` files — prose is kept minimal, syntax
and examples are verbatim or closely modeled on the docs. Source pages are noted per section;
if a page 404'd or had no examples, that is called out explicitly rather than inventing syntax.

---

## Lexical Structure

Source: `language-reference/lexical-structure`.

All keywords begin with `\`. Full list:

```
\open \import \hiding \as \using
\truncated \data \cons \func \lemma \axiom \sfunc \type \class \record \meta
\classifying \noclassifying \strict \alias
\field \property \override \default \extends \module \instance
\use \coerce \level \levels \plevels \hlevels \box \eval \peval
\with \elim \cowith \where
\infix \infixl \infixr \fix \fixl \fixr
\new \this \Pi \Sigma \lam \let \let! \have \have! \in \case \scase \return
\lp \lh \suc \max \oo
\Prop \Set \Type
```

**Numerals**: a positive numeral is a non-empty sequence of digits; a negative numeral is `-`
followed by a non-empty sequence of digits.

**Identifiers**: a non-empty sequence of upper/lower-case letters, digits, and characters from
`` '~!@#$%^&*-+=<>?/|[]:_ ``, excluding sequences starting with a digit or `'`, numerals, and the
reserved names `->`, `=>`, `_`, `:`, `|`.

- Valid: `xxx`, `+`, `$^~]!005x`, `::`, `->x`, `x:Nat`, `-5b`, `-33+7`, `--xxx`, `f'`
- Invalid: `5b`, `-33`, `=>`, `α`

**Infix/postfix notation**: a postfix notation is an identifier following `` ` ``; an infix
notation is an identifier surrounded by `` ` ` ``  (see Definitions/Precedence below).

**Comments**:

```ard
{- multi-line comment, can be {- nested -} -}
-- a single-line comment needs >= 2 dashes followed by whitespace
------- also a valid comment
--foo    -- INVALID: no whitespace after dashes
foo--bar -- INVALID
```

---

## Definitions: Precedence, Infix Operators, Aliases

Source: `language-reference/definitions`.

Every definition (function, data, record, class, instance, coercion) has a name, and is invoked
via a **defcall**: `def e_1 … e_n`. With exactly two arguments you may write infix:
`e_1 \`def\` e_2`. With exactly one argument you may write postfix: `e_1 \`def`.

### Precedence

```ard
\func \fixl 3 op (a b : Nat) => 0
```

`\fixl`/`\fixr`/`\fix` mark a definition as left-, right-, or non-associative, followed by a
priority `N` (integer 1–9). Default precedence is `\fixr 10`.

Parsing rule for `e1 \`op1\` e2 \`op2\` e3`: higher priority binds tighter; equal priority with
matching associativity groups accordingly; equal priority with mismatched/non-associativity is a
parse error.

### Infix operators

```ard
\infixl 6 +
```

Declares `+` so defcalls parse infix even without `` ` ``. It can still be used prefix: `+ 1 2`
same as `1 + 2`. Sections:

```ard
x +        -- \lam y => x + y   (left section)
`+ y       -- \lam x => x + y   (right section)
```

### Aliases

```ard
\func \infixl 6 +  \alias plus (a b : Nat) : Nat => ...
```

`\alias aliasName` after a definition's name gives an interchangeable alternative name; it can
carry its own precedence and may use unicode symbols in U+2200–U+22FF.

---

## Modules and `\where` blocks

Source: `language-reference/definitions/modules`.

### Modules

```ard
\module Mod \where {
  def_1
  ...
  def_n
}

-- single definition: braces optional
\module Mod \where
  \func f1 => 0
```

```ard
\func f2 => Mod.f1
\module Mod \where {
  \func f1 => f2   -- refers to Mod.f2, NOT top-level f2
  \func f2 => 0
  \func f3 => f4   -- refers to top-level f4
}
\func f4 => f2
```

### `\where` blocks

Every definition has an associated module of the same name; `\where` attaches local definitions,
visible inside the parent definition:

```ard
\func f => g \where \func g => 0

\func h => f.g \where
  \data D \where {
    \func k => D
    \func s => M.g.N.s
  }
\module M \where
  \func g => N.s \where {
    \module N \where {
      \func s => E
    }
    \data E
  }
```

Constructors of `\data` and fields of `\class`/`\record` live in the associated module but are
also visible outside it:

```ard
\data d | a | b d

\func f1 => b a
\func f2 => d.b a   -- equivalent to f1
```

`\where` blocks of `\data`/`\class` may contain special `\use` instructions (`\use \coerce`,
`\use \level`) that modify the parent definition — see Coercion and Level sections below.

Parameters of the enclosing definition are visible in its `\where` block; if used there, they and
their dependencies are threaded automatically when called from within the block, but must be
passed explicitly from outside:

```ard
\func f (x : Nat) (p : x = 0) => p
  \where {
    \func g => p
    \func h : x = 0 => g
  }
\func g : 0 = 0 => f.g 0 idp
\func h : 0 = 0 => f.h 0 idp
```

### Open commands

```ard
\open M                          -- everything from M
\open M (def_1, def_2)           -- only these, others need full name
\open M \hiding (def_1, def_2)   -- everything except these
\open M (def_1 \as def_1', def_2 \as def_2')          -- rename on open
\open M \using (def_1 \as def_1', def_2 \as def_2')   -- open all, renaming some
```

```ard
\module M \where {
  \func f => 0
  \func g => 1
  \func h => 2
}
\module M1 \where {
  \open M (f,g)
  \func h1 => f
  \func h2 => g
  \func h3 => M.h        -- only accessible via full name
}
\module M2 \where {
  \open M \hiding (f,g)
  \func h1 => M.f        -- only accessible via full name
  \func h2 => M.g
  \func h3 => h
}
\module M3 \where {
  \open M1 (h1 \as M1_h1, h2)
  \open M2 \using (h2 \as M2_h2) \hiding (h3)
  \func k1 => M1_h1      -- M1.h1
  \func k2 => h1         -- M2.h1
  \func k3 => h2         -- M1.h2
  \func k4 => M2_h2      -- M2.h2
  \func k5 => M1.h3      -- only via full name
}
```

Opening `M` inside `M'`, then opening `M'` inside `M''`, does **not** transitively expose `M`'s
contents in `M''`.

### Import commands

```ard
-- A.ard
\func a1 => 0
\func a2 => 0
  \where \func a3 => 0

-- Dir/C.ard
\import A
\func c1 => a1
\func c2 => a2.a3

-- B.ard
\import Dir.C
\func b1 => c1
-- \func b2 => a1          -- NOT visible: A wasn't imported by B
-- \func b3 => A.a1        -- NOT visible either
\func b4 => Dir.C.c2        -- fully-qualified access always works
```

`\import` also opens the file's contents (same `\open`-style modifiers apply). To import without
opening anything:

```ard
-- X.ard
\func f => 0

-- Y.ard
\import X()
\func f => X.f
```

---

## Parameters

Source: `language-reference/definitions/parameters`.

Four forms: `(x : T)` typed-named-explicit, `T` typed-unnamed-explicit (data/constructors, Pi/Sigma
only), `{x : T}` typed-named-implicit, `{T}` typed-unnamed-implicit.

```ard
\data d
  | c1 (a : Nat)   -- typed named explicit
  | c2 Nat         -- typed unnamed explicit
  | c3 {a : Nat}   -- typed named implicit
  | c4 {Nat}       -- typed unnamed implicit
  -- | c5 b         -- untyped parameters NOT allowed in constructors

\func g1 => \lam a => suc a     -- untyped named explicit lambda param OK
\func g2 => \lam {a} => suc a   -- untyped named implicit lambda param OK
-- \func f a => a                -- INVALID: func params must be named AND typed
```

Grouping same-typed parameters:

```ard
\func f {x1 x2 : A1} (y1 y2 y3 : A2) (z : A3) => B
-- equivalent to:
\func f {x1 : A1} {x2 : A1} (y1 : A2) (y2 : A2) (y3 : A2) (z : A3) => B
```

Unused parameter name can be `_`.

### Strict parameters

```ard
\func f (n : Nat) (\strict m k : Nat) (l : Nat) => n
```

`m` and `k` are evaluated immediately on invocation (perf only, no semantic change).

### Implicit arguments

```ard
\func f (x : A1) {y y' : A2} (z : A3) {z : A4} => 0
\func g => f a1 a3            -- y, y', last z inferred
\func g => f _ a3             -- explicitly ask the typechecker to infer x
\func g => f _ {a2} a3 {a4}   -- x, y' still implicit-inferred; y and last z given explicitly
```

Implicit args right after an infix operator bind to its first parameters:

```ard
\func f (A : \Type) => \lam a b => a = {A} b
```

An implicit argument can be inferred automatically if its type has a definitionally unique
element (a no-parameter `\Sigma`-type or a record with no fields).

---

## Functions

Source: `language-reference/definitions/functions`.

Signature: `\func f p_1 … p_n : T`. Result type can often be omitted when inferable from the body.

### Non-recursive definitions

```ard
\func id (x : A) => x
\func second (x : A) (y : B) (z : C) => y

-- equivalent, with explicit result type:
\func id (x : A) : A => x
\func second (x : A) (y : B) (z : C) : B => y
```

### Pattern matching

```ard
\func f (d : D) : Nat
  | con1 => 0
  | con2 n => suc n
```

General form:

```ard
\func f (x_1 : T_1) ... (x_n : T_n) : R
  | clause_1
  ...
  | clause_k
```

Result type must be given explicitly. Clauses must be exhaustive over the constructors of the
matched types. Every clause must match **all** `n` parameters (unless using `\elim`, below).
Pattern matching on `left`/`right` of `I` is **not allowed** in ordinary functions.

Implicit parameter patterns must be omitted or wrapped in `{ }`.

`\with` is an alternative to bare `|` clauses:

```ard
\func f p_1 ... p_n : R \with { | clause_1 ... | clause_k }
```

**Pattern forms**: a variable (or `_` if unused); `con s_1 … s_m` for constructor `con`; a tuple
`(s_1, … s_m)` matching a Sigma-type/record/class; an as-pattern `pat \as x : E`.

**Evaluation / matching order** — patterns are tried left-to-right, top-to-bottom, and a call does
not reduce until a full match is found (not merely "compatible with the first clause"):

```ard
\func g (b b' : B) : Nat
  | T, _ => 0
  | _, T => 1
  | _, _ => 2

g T e => 0
g x e     -- does not reduce (x is a free variable, not T or F)
g F T => 1
g F F => 2
g F x     -- does not reduce
```

```ard
\func \infix 4 < (n m : Nat) : Bool
  | _, 0 => false
  | 0, suc _ => true
  | suc n, suc m => n < m
```

The funcall `n < 0` (`n` a free/bound variable, not yet a known constructor shape) does **not**
evaluate: since some other clause needs to inspect the first argument, the matcher must inspect
argument 1 before it can commit to clause 1, and a bare variable there is stuck — even though
clause 1's first pattern is just a wildcard. By contrast `0 < 0` and `suc n < 0` **do** evaluate to
`false`, because in both cases argument 1 is already a known constructor form (`0` or `suc n`),
so inspecting it doesn't get stuck, and clause 1 (`_, 0`) matches.

**Absurd pattern** `()` for matching on an empty type — right-hand side must be omitted:

```ard
\data Empty

\func absurd {A : \Type} (e : Empty) : A
  | ()

-- can often be omitted entirely:
\func absurd {A : \Type} (e : Empty) : A
```

`con1 = con2` is empty whenever `con1`/`con2` are disjoint constructors of a data type.

Infix constructors work in patterns either way:

```ard
\data List (A : \Type) | nil | \infixr 5 :: A (List A)

\func tail {A : \Type} (l : List A) : List A
  | nil => nil
  | :: _ l => l

\func tail' {A : \Type} (l : List A) : List A
  | nil => nil
  | _ :: l => l
```

### `\elim`

Use `\elim` to pattern-match only on a subset of parameters, leaving the rest untouched/visible:

```ard
\func f (x_1 : A_1) ... (x_n : A_n) : R \elim x_{i_1}, ... x_{i_m}
  | p^1_1, ... p^1_m => e_1
  ...
  | p^k_1, ... p^k_m => e_k
```

```ard
\func if (b : B) (t e : X) : X \elim b
  | T => t
  | F => e
```

The eliminated params are not visible in the RHS; other params still are. Explicit vs. implicit
doesn't matter for eliminated params — patterns for them are always written explicitly.

### Recursive functions and mutual recursion

Recursive calls must be **structural**: arguments to recursive calls must be subpatterns of `f`'s
own parameter patterns (checked by the termination checker, see its own section). Mutually
recursive function groups must admit a linear order where each signature only refers to earlier
ones; bodies may refer to each other as long as the whole system is structural.

### Copattern matching (`\cowith`)

When the result type is a record/class:

```ard
\func f (x_1 : A_1) ... (x_n : A_n) : C \cowith
  | coclause_1
  ...
  | coclause_k
```

Each `coclause` is `field => expr`. Equivalent to:

```ard
\func f (x_1 : A_1) ... (x_n : A_n) => \new C {
  | coclause_1
  ...
  | coclause_k
}
```

### Patterns in coclauses

A field implementation can itself be pattern-matched:

```ard
\func f (x_1 : A_1) ... (x_n : A_n) : C \cowith
  ...
  | field (y_1 : B_1) ... (y_k : B_k) : D \with {
    | clause_1
    ...
    | clause_m
  }
  ...
```

`\with` may be replaced by `\elim`. Such clauses currently cannot refer to `f`'s own parameters.

### Lemmas and axioms

```ard
\lemma lemma_name : PropositionType => proof
\axiom axiom_name : Type   -- a lemma without a body
```

A `\lemma`'s result type must be a proposition (or provably a proposition via `\level`, see
Level section); its body is erased/does-not-evaluate, which speeds up typechecking of large
proof terms.

### `\sfunc`, `\eval`, `\peval`

```ard
\sfunc f a_1 ... a_n => body   -- does not evaluate, like a lemma
\eval f a_1 ... a_n             -- forces evaluation of the \sfunc call
\peval f a_1 ... a_n             -- : (f a_1 ... a_n = \eval f a_1 ... a_n)
```

---

## Type synonyms (`\type`)

Source: `language-reference/definitions/types`.

```ard
\type T p_1 ... p_n => E

\type Pair (A : \Type) => \Sigma A A
```

A type synonym is a function that returns a type and **does not evaluate** — `Pair A` and
`\Sigma A A` are type-equivalent but not computationally equivalent. Coercion between them is
implicit:

```ard
\func coerceFrom (p : Pair Nat) : \Sigma Nat Nat => p
\func coerceTo (p : \Sigma Nat Nat) : Pair Nat => p
```

---

## Data types (`\data`)

Source: `language-reference/definitions/data`.

```ard
\data D p_1 ... p_n
  | con_1 p^1_1 ... p^1_{k_1}
  ...
  | con_m p^m_1 ... p^m_{k_m}

-- with explicit universe:
\data D p_1 ... p_n : \h-Type p
  | con_1 p^1_1 ... p^1_{k_1}
  ...
```

Parameters `p_1 … p_n` of `D` become **implicit** parameters of every constructor:

```ard
\data Data A B
  | cons {C} D E
  | cons'

\func s1 => Data a b
\func s2 => cons d e
\func s2' => cons {a} d e
\func s2'' => cons {a} {b} {c} d e
\func s3 => cons'
\func s3' => cons' {a}
\func s3'' => cons' {a} {b}
```

Constructors live in `D`'s associated module, but are also visible where `D` itself is defined:

```ard
\data D | con1 | con2

\func f => con1
\func g => con2
\func f' => D.con1   -- same as f
\func g' => D.con2   -- same as g
```

### Inductive definitions / strict positivity

`D` may recursively occur inside constructor parameter types (not in `D`'s own parameters
`p_1…p_n`), but only in **strictly positive** positions:

- `D` occurs strictly-positively in `D a_1 … a_n` if absent from `a_1…a_n`.
- in `\Pi (x : A) -> B` if strictly positive in `B` and absent from `A`.
- in `Path (\lam i => B) b b'` if strictly positive in `B` and absent from `b`, `b'`.
- in anything else, only if entirely absent.

### Truncation

```ard
\truncated \data D p_1 ... p_n : \h-Type p
  | con_1 p^1_1 ... p^1_{k_1}
  ...
```

Truncates to a homotopy level below `D`'s actual level; the typechecker errors if `D`'s actual
predicative level exceeds `p`. A truncated data type can only be eliminated into types of the
same-or-lower homotopy level:

```ard
\truncated \data Exists (A : \Type) (B : A -> \Type) : \Prop
  | witness (a : A) (B a)

-- INVALID — Nat is a Set, not a Prop, so it cannot be extracted from a truncated \Prop:
-- \func extract (p : Exists Nat (\lam n => n = 3)) : Nat
--   | witness a b => a

-- VALID — result types are themselves propositions:
\func existsSuc (p : Exists Nat (\lam n => n = 3)) : Exists Nat (\lam n => suc n = 4)
  | witness n p => witness (suc n) (path (\lam i => suc (p @ i)))

\func existsEq (p : Exists Nat (\lam n => n = 3)) : 0 = 0
  | witness n p => path (\lam _ => 0)
```

If the target universe is bigger than `D`'s but provably fits `D`'s actual level, use `\level`
(see Level section) and declare the function as `\sfunc` (truncated data is "squashed").

### Induction-induction / induction-recursion

Two-or-more mutually recursive `\data` (induction-induction) or `\data` mutually recursive with
`\func` (induction-recursion) must satisfy the same strict-positivity and termination rules as
ordinary recursive definitions.

### Varying number of constructors (`\elim`/`\with` on data)

```ard
\data Vec (A : \Type) (n : Nat) \elim n
  | 0 => nil
  | suc n => cons A (Vec A n)
```

General form (`\elim` allows matching a subset of `D`'s params; `\with` requires matching all):

```ard
\data D p_1 ... p_n \with
  | t^1_1, ... t^1_n => con_1 p^1_1 ... p^1_{k_1}
  ...
  | t^m_1, ... t^m_n => con_m p^m_1 ... p^m_{k_m}
```

Clause order doesn't matter; whichever clause matches a given `D a_1 … a_n` contributes its
constructor:

```ard
\data Bool | true | false

\data T (b : Bool) \with
  | true => con1
  | true => con2
-- T true has constructors con1 AND con2; T false is empty (no matching clause)

-- multiple constructors under one clause, equivalently:
\data T (b : Bool) \with
  | true => {
    | con1
    | con2
  }
```

### Constructor synonyms (`\cons`)

```ard
\cons f p_1 ... p_n => e
```

A function defined with `\cons` cannot be defined by pattern matching, must consist only of
constructor applications, and can be used inside patterns as a synonym for its RHS.

---

## Higher Inductive Types (HITs)

Source: `language-reference/definitions/hits`.

### Conditions

A **condition** on constructor `con` is a rule (defined via pattern matching on `con`'s own
parameters) for how `con a_1 … a_n` reduces further. Unlike ordinary pattern matching, conditions
need not be exhaustive, and matching on `left`/`right` of `I` **is** allowed here.

```ard
\data Int
  | pos Nat
  | neg Nat \with {
    | zero => pos zero
  }
```

Any function out of a conditioned data type must respect the conditions (checked by the
typechecker) — e.g. a function `Int -> X` must send `pos zero` and `neg zero` to the same value,
so this does **not** typecheck:

```ard
-- INVALID: neg zero and pos zero must map to the same value, but this maps them to 0 and 1
\func f (x : Int) : Nat
  | pos n => n
  | neg n => suc n
```

### HITs proper (interval-valued conditions)

A HIT is a data type with a constructor whose conditions are of the form `| left => e | right => e'`:

```ard
-- Circle
\data S1
  | base
  | loop I \with {
    | left => base
    | right => base
  }

-- Suspension
\data Susp (A : \Type)
  | north
  | south
  | merid A (i : I) \elim i {
    | left => north
    | right => south
  }

-- Propositional truncation
\data Trunc (A : \Type)
  | inT A
  | truncT (x y : Trunc A) (i : I) \elim i {
    | left => x
    | right => y
  }

-- Set quotient
\data Quotient (A : \Type) (R : A -> A -> \Type)
  | inQ A
  | equivQ (x y : A) (R x y) (i : I) \elim i {
    | left => inQ x
    | right => inQ y
  }
  | truncQ (a a' : Quotient A R) (p p' : a = a') (i j : I) \elim i, j {
    | i, left  => p @ i
    | i, right => p' @ i
    | left,  _ => a
    | right, _ => a'
  }
```

To define `Trunc A -> X` where `X` is a proposition, it suffices to give the value on `inT a`
only — likewise for any HIT eliminating into a type at or below its "natural" homotopy level;
e.g. for `Quotient A R -> X` with `X` a set you must handle `inQ a` and `equivQ x y r i`, but if
`X` is merely a proposition, `inQ a` alone suffices.

### Alternate path-typed syntax for HITs

Instead of writing `I`-conditions explicitly, give the constructor's type as an iterated path type
in the data type itself — its endpoints become the conditions:

```ard
\data S1 | base | loop : base = base

\data Susp (A : \Type)
  | north
  | south
  | merid A : north = south

\data Trunc (A : \Type)
  | inT A
  | truncT (x y : Trunc A) : x = y

\data Quotient (A : \Type) (R : A -> A -> \Type)
  | inQ A
  | equivQ (x y : A) (R x y) : inQ x = inQ y
  | truncQ (a a' : Quotient A R) (p p' : a = a') : Path (\lam i => p i = p' i) idp idp
```

### Pattern matching on HITs

Interval variables can be omitted in the RHS when it is a path — the typechecker inserts them and
applies the RHS:

```ard
\func double (x : S1) : S1
  | base => base
  | loop => loop *> loop        -- sugar for:

\func double' (x : S1) : S1
  | base => base
  | loop i => (loop *> loop) i
```

(`*>` is path concatenation, from the standard library, not the prelude.)

---

## Records

Source: `language-reference/definitions/records`.

A record is a Sigma type with named projections.

```ard
\record R (p_1 : A_1) ... (p_n : A_n) {
  | f_1 : B_1
  ...
  | f_k : B_k
}
```

`p_1…p_n` and `f_1…f_k` are all fields of `R`. The only difference: `f_j` are in scope *inside*
`R`'s own definition, `p_i` are not.

```ard
\func test1 => f_1        -- OK, f_1 is in scope in R's module
\func test2 => R.f_1
\func test3 => R.p_1
-- \func test4 => p_1     -- INVALID: p_1 not in scope by itself
```

`\field f_i : B_i` is equivalent to `| f_i : B_i` except for the `\property` distinction (below).

### Field access

```ard
\func test5 (x : R) => R.p_1 {x}
\func test6 (x : R) => f_1 {x}

-- dot syntax — only when the LHS is a variable with an explicit record type:
\func test5' (x : R) => x.p_1
\func test6' (x : R) => x.f_1
```

### Creating instances (`\new`)

```ard
\func r1 => \new R a_1 ... a_n { | f_1 => b_1 ... | f_k => b_k }
\func r2 => \new R { | p_1 => a_1 ... | p_n => a_n | f_1 => b_1 ... | f_k => b_k }
\func r3 => \new R a_1 ... a_n b_1 ... b_k
\func r4 => \new R a_1 ... a_i { | p_{i+1} => a_{i+1} ... | p_n => a_n | f_1 => b_1 ... | f_k => b_k }
\func r5 => \new R a_1 ... a_n b_1 ... b_i { | f_{i+1} => b_{i+1} ... | f_k => b_k }
```

Or via copattern matching:

```ard
\func r6 : R \cowith
  | p_1 => a_1
  ...
  | p_n => a_n
  | f_1 => b_1
  ...
  | f_k => b_k
```

Records satisfy the eta rule: `\new R r.p_1 … r.p_n r.f_1 … r.f_k` is equivalent to `r`.

### `\field` vs `\property`

```ard
\record NegativeInt {
  \field x : Int
  \property isNeg : x < 0
}
```

A `\property`'s type must be a proposition, and (like a `\lemma`) it does **not evaluate**:

```ard
\func test (x : Int) (p : x < 0) => isNeg {\new NegativeInt x p}
-- test x p does NOT reduce to p (isNeg is a property)
```

Note: if a plain `| f : A` field's type `A` already is a `\Prop`, it is automatically treated as a
property; write `\field f : A` explicitly to force a non-property field of a propositional type.

### Extensions (`\extends`)

```ard
\record S (r_1 : D_1) ... (r_t : D_t) \extends R {
  | g_1 : C_1
  ...
  | g_m : C_m
  | p_{i_1} => a_{i_1}
  ...
  | f_{j_1} => b_{j_1}
  ...
}
```

`S` is a subtype of `R`. A record is equivalent to the Sigma type of its unimplemented fields:

```ard
\record C (x y : Nat) {
  | x<=y : x <= y
  | y<=0 : y <= 0
}

\record D \extends C {
  | y => x
  | x<=y => <=-reflexive x
}
-- D is equivalent to \Sigma (x : Nat) (x <= 0):

\func fromD (d : D) : \Sigma (x : Nat) (x <= 0) => (d.x, d.y<=0)
\func toD (p : \Sigma (x : Nat) (x <= 0)) => \new D p.1 p.2
\func fromToD (d : D) : toD (fromD d) = d => idp
\func toFromD (p : \Sigma (x : Nat) (x <= 0)) : fromD (toD p) = p => idp
```

Multiple inheritance — shared base fields aren't duplicated; distinct same-named fields need
fully-qualified access:

```ard
\record A (x : Nat)
\record B \extends A
\record C \extends A
\record D \extends B,C     -- D has a SINGLE field x

\record B (x : Nat)
\record C (x : Nat)
\record D \extends B,C     -- B.x and C.x are DIFFERENT fields here

\func fromD (d : D) : \Sigma Nat Nat => (B.x {d}, C.x {d})
\func toD (p : \Sigma Nat Nat) => \new D p.1 p.2
```

### `\this`

Every field of `R` implicitly takes a parameter of type `R`, referenceable as `\this`:

```ard
\record R (X : \Type) (t : X -> X)

\func f (r : R) => r.t

\record S \extends R {
  | x : X
  | p : f \this x = x
}
```

`\this` may appear only in arguments of definitions that themselves satisfy this same condition.

### Dynamic definitions

A `\func`/`\data` nested directly inside a `\record` body gets an implicit `R`-typed parameter
automatically, accessible via dot-syntax:

```ard
\record R {
  | n : Nat
  \func f (x : Nat) => x Nat.+ n
}
-- equivalent to:
\record R {
  | n : Nat
} \where {
  \func f {this : R} (x : Nat) => x Nat.+ this.n
}

\func g (r : R) : Nat => r.f 1
```

### `\override`

```ard
\record R
\record S \extends R

\record A {
  | f : R
}

\record B \extends A {
  \override f : S
}
```

### `\default`

Default implementations apply only when an instance omits the field:

```ard
\record R
  | f : Nat
  | g : Nat

\record S \extends R {
  \default f => 0
}

\func inst1 : S \cowith
  | f => 1
  | g => 2

\func inst2 : S \cowith
  | g => 2

\func test1 : inst1.f = 1 => idp
\func test2 : inst2.f = 0 => idp
```

Defaults may pattern-match; the generated helper function can be renamed with `\as`:

```ard
\record S \extends R {
  \default f (n : Nat) : Nat \with {
    | 0 => 0
    | suc n => n
  }
}

\record S' \extends R {
  \default f \as fImpl (n : Nat) : Nat \with {
    | 0 => 0
    | suc n => n
  }
}
```

Defaults don't see each other unless named/typed explicitly, or can be mutually defined:

```ard
-- does NOT work: second default can't see the first's value
\record S \extends R {
  \default x => 0
  \default p => idp   -- fails
}

-- fixed via naming + explicit type:
\record S' \extends R {
  \default x \as x' => 0
  \default p : x' = 0 => idp
}

-- mutual defaults; instantiator must fix at least one:
\record R
  | x : Nat
  | y : Nat

\record S \extends R {
  \default x => y
  \default y => x
}

\func test1 : S \cowith | x => 0
\func test2 : S \cowith | y => 1
\func test3 : S \cowith | x => 2 | y => 3
```

### Levels on `\extends`

```ard
\record R
  | A : \Type
  | B : \1-Type
  | C : \Type1

\record S \extends R (0,0)
-- for s : S:  s.A : \Set0,  s.B : \1-Type0,  s.C : \Set1
```

---

## Classes and Instances

Source: `language-reference/definitions/classes`.

A `\class` is a `\record` with extra powers: it can extend/be-extended-by other classes and
records, and its **first explicit parameter is its classifying field** — classes implicitly
coerce to that field's type.

```ard
\class Semigroup (E : \Set)
  | \infixl 7 * : E -> E -> E
  | *-assoc (x y z : E) : (x * y) * z = x * (y * z)

\func square {S : Semigroup} (x : S) => x * x   -- x : S.E via implicit coercion
```

A parameter whose type is an extension `C { … }` of class `C` is a **local instance** of `C`
(as is the implicit `\this` of a field of `C`). Instance inference searches local instances for
one whose classifying-field value matches the expected value.

### Explicit / absent classifying field

```ard
\class C (A : \Set) (\classifying B : \Set)
  | f : B -> A

\func test {c : C} (x : c) => f x   -- x : c.B here, since B is \classifying
```

```ard
\class ClassName \noclassifying (explicit params...)
```

`\noclassifying` (written after the class name) defines a class with no classifying field, for
classes that do have explicit parameters but none of them should classify.

### Global instances (`\instance`)

```ard
\instance NatSemigroup : Semigroup Nat
  | * => Nat.+
  | *-assoc (x y z : Nat) : (x Nat.+ y) Nat.+ z = x Nat.+ (y Nat.+ z) \elim z {
    | 0 => idp
    | suc z => pmap suc (*-assoc x y z)
  }
```

An `\instance` is just a function defined by copattern matching, usable as an ordinary function,
except it *must* be defined via copattern matching and must produce a class instance; and its
classifying field must be a data/record applied to arguments (if a parameter of the instance is
itself class-typed, its classifying field's value must be an argument of the instance's own
classifying field).

### Instance resolution

If no local instance matches, global instances are searched; **no backtracking** — the first
matching global instance is taken. A global instance matches if its classifying field is the
*same* data/record as expected, regardless of the arguments applied to it.

---

## Meta definitions (`\meta`)

Source: `language-reference/definitions/metas`. (Thin page — most metas are defined in Java; this
is the only in-language form documented, with no further examples given.)

```ard
\meta f x_1 ... x_n => e
```

`e` is typechecked only when `f` is invoked; `f e_1 … e_n` is replaced by `e[e_1/x_1,…,e_n/x_n]`
and *that* is typechecked in place of the original call.

---

## Coercion (`\use \coerce`)

Source: `language-reference/definitions/coercion`.

A coercing function `f : A -> B` lets an `a : A` be used silently wherever `B` is expected —
`a` is auto-replaced by `f a`. Must be declared inside the `\where` block of a `\data` or `\class`
definition, using `\use \coerce` instead of `\func`.

**Coerce to a type** (function's result type is the definition):

```ard
\data Bool | true | false
  \where
    \use \coerce toNat (b : Bool) : Nat
      | true => 1
      | false => 0
```

**Coerce from a type** (function's *last parameter* type is the definition):

```ard
\data Bool | true | false
  \where
    \use \coerce fromNat (n : Nat) : Bool
      | 0 => false
      | 1 => true
      | suc (suc n) => fromNat n
```

Multiple coercing functions may be declared for one type. If a type coerces to a `\Pi`-type
(function type), its elements become directly applicable — the coercion is inserted automatically.

### Coercing fields/constructors

```ard
\record R (\coerce A : \Type) (a : A)
\data D | \coerce con1 Nat | con2
```

A field/constructor marked `\coerce` acts as a coercing function itself.

---

## Level (`\use \level`, homotopy-level proofs)

Source: `language-reference/definitions/level`.

### `\use \level` on `\data`/`\class`/`\record`

Proves a definition belongs to a smaller homotopy-level universe than automatically inferred.
Written in the `\where` block, replacing `\func` with `\use \level`:

```ard
\data Empty
\data Dec (P : \Prop) | yes P | no (P -> Empty)
  \where
    \use \level isProp {P : \Prop} (d1 d2 : Dec P) : d1 = d2
      | yes x1, yes x2 => path (\lam i => yes (Path.inProp x1 x2 @ i))
      | yes x1, no e2 => \case e2 x1 \with {}
      | no e1, yes x2 => \case e1 x2 \with {}
      | no e1, no e2 => path (\lam i => no (\lam x => (\case e1 x \return e1 x = e2 x \with {}) @ i))
```

Such a function's leading parameters must be the parent's own parameters/fields; the rest, plus
result type, must prove `ofHLevel_-1+ (D p_1 … p_k) n` for a fixed `n`, where:

```ard
\func \infix 2 ofHLevel_-1+ (A : \Type) (n : Nat) : \Type \elim n
  | 0 => \Pi (a a' : A) -> a = a'
  | suc n => \Pi (a a' : A) -> (a = a') ofHLevel_-1+ n
```

### `\level` in a result type

Used when a lemma/property/case's natural result type doesn't fit its universe, but a proof of the
right homotopy level exists. First argument is the type, second is the leveling proof:

```ard
\lemma lem : \level A p => {?}

\record R {
  \property s : \level A p
}

\func f (t : Trunc A) : \level B p
  | inT a => g a

\func f' (t : Trunc A) => \case t \return \level B p \with {
  | inT a => g a
}
```

### `\use \level` on a `\func`

Doesn't change the function's type at all — pure sugar that lets the typechecker treat the
function's result as living in a lower universe wherever "level of a type" matters:

```ard
\func isProp (A : \Type) => \Pi (a a' : A) -> a = a'
  \where \use \level proof (A : \Type) : isProp (isProp A) => {?}

\lemma lem : isProp (\Sigma (n : Nat) (n = 0)) => {?}
```

### Squashed data types

A data type is "squashed" if `\truncated` or annotated with `\use \level`. Pattern matching on a
squashed type is only allowed if the result universe is `<=` the data type's own universe, or the
match happens inside `\sfunc`, `\lemma`, `\use \level`, or `\scase`.

---

## Expressions: General

Source: `language-reference/expressions`.

Every expression has a type (`e : E`); `(e : E)` (parens required) is an explicit type
annotation. A **defcall** `f a_1 … a_n` applies definition `f` (n params) to args. Partial
application `f a_1 … a_k` (`k < n`) is valid and equals `\lam a_{k+1} … a_n => f a_1 … a_n`.

`=>` (the reduction relation) defines normal forms; `==` (computational equality) is its
reflexive/symmetric/transitive closure — the typechecker compares expressions by normalizing
both sides.

---

## Universes

Source: `language-reference/expressions/universes`.

Predicative hierarchy `\Type n` (whitespace optional), cumulative (`\Type n` values also have
type `\Type (n+1)`). Types are further stratified by homotopy level `h` (`-1 <= h <= oo`) into
`\h-Type p`. Named universes:

- `\Prop` — propositions, `-1`-types, **impredicative** (no predicative level): if `B : \Prop`
  then `\Pi (x : \Prop) -> B : \Prop`. Not proof-irrelevant in general, but elements that never
  reduce to a constructor are computationally equal (e.g. any two elements of an empty type).
- `\Set n` — sets, `0`-types, same as `\0-Type n`.
- `\oo-Type p` — infinite homotopy level.
- `\Type p h` is equivalent to `\h-Type p`.

### Placement rules

- `A : \h1-Type p1`, `B : \h2-Type p2` ⟹ `\Sigma A B : \max(h1,h2)-Type max(p1,p2)`.
- `A : \h1-Type p1`, `B : \h2-Type p2` ⟹ `\Pi (x:A) -> B : \h2-Type max(p1,p2)` (Pi's homotopy
  level is just the codomain's).
- `0 <= h < oo` ⟹ `\h-Type p : (h+1)-Type (p+1)`.
- `\Prop : \Set0`.
- `\oo-Type p : \oo-Type (p+1)`.
- `A : I -> \h-Type p` ⟹ `Path A a a' : \max(-1,h-1)-Type p`.
- For data type `D` with constructor-parameter types of level `(h_i, p_i)`: predicative level is
  `max(0, p_1,…,p_k)`. If `D` has interval-conditions equating endpoints, homotopy level is `oo`.
  Else, homotopy level is `max(0, h_1,…,h_k)` if `D` has >1 constructor, or `max(-1, h_1,…,h_k)`
  if `D` has <=1 constructor.
- For class/record `C` with unimplemented-field types of level `(h_i, p_i)`: predicative level
  `max(0,p_1,…,p_k)`, homotopy level `max(-1,h_1,…,h_k)`.

### Level polymorphism

Every definition is implicitly polymorphic in a predicative level `\lp` and homotopy level `\lh`.

```ard
\func id {A : \Type} (a : A) => a
-- equivalent, with explicit levels:
\func id' {A : \Type \lp \lh} (a : A) => a
```

Explicit level args in a defcall: `\levels p h` (often omittable):

```ard
\data Either (A B : \Type) | inl A | inr B
\data Either' (A B : \Type \lp \lh) | inl A | inr B

\func f => id \Type
\func f' => id (\suc \lp) (\suc \lh) (\Type \lp \lh)

\func fromEither {A : \Type} (e : Either A \Type) : \Type \elim e
  | inl a => A
  | inr X => X
```

Level expressions: `\lp`/`\lh` (variables), a natural-number constant (`\oo` also valid for
homotopy), `_` (infer), `\suc l`, `\max l1 l2`. `\levels \Prop` picks the Prop level.

### Multiple level parameters

```ard
\func func \plevels p1 <= p2 <= p3 \hlevels h1 >= h2 (A : \Type p2 h1) (B : \Type p3 h2) => \Type p1
\func example => func \levels (1,2,3) (2,1) Nat Nat
```

Same-kind level params must be linearly ordered. If not declared explicitly, level params are
inherited from used definitions when all agree.

Global declarations (usable by any definition that opts in by referencing them):

```ard
\plevels p1 <= p2 <= p3
\hlevels h1 >= h2
```

### Level inference gotchas

The typechecker infers the *minimal* level mentioning `\lp`/`\lh` where possible:

```ard
\func id {A : \Type} (a : A) => a
```

Levels in the **signature** of a recursive function are inferred *before* levels in the body —
this can fail:

```ard
-- FAILS to typecheck: body's \Type needs a level not fixed by the signature
\func eitherToType {A : \Type} (e : Either A A) : \Type
  | inl _ => \Type
  | inr _ => \Type

-- fix: pin the result-type universe higher than the parameter universe
\func eitherToTypeFixed {A : \Type} (e : Either A A) : \Type (\suc \lp) (\suc \lh)
  | inl _ => \Type
  | inr _ => \Type
```

Homotopy levels the checker infers are always `>= 0`, so this also fails:

```ard
-- FAILS: inferred homotopy level can't be \Prop's -1
\func eitherToProp {A : \Type} (e : Either A A) : \Set0
  | inl _ => \Type
  | inr _ => \Type

-- fix by fixing the level explicitly to a constant that allows \Prop:
\func eitherToPropFixed {A : \Type} (e : Either A A) : \Set0
  | inl _ => \Prop
  | inr _ => \Prop
```

Levels in the result type of a **non-recursive** function are inferred jointly with the body, so
this works fine:

```ard
\func f : \Type => \Type
```

---

## Pi Types and Lambda

Source: `language-reference/expressions/pi`.

```ard
\Pi p_1 … p_n -> B          -- dependent function type
A_1 -> … -> A_n -> B        -- non-dependent sugar, when B doesn't mention the params
```

If `A_i : \Type p_i h_i` and `B : \Type p h`, then `\Pi p_1…p_n -> B : \Type p_max h` (homotopy
level of a Pi type = homotopy level of the codomain only).

Lambda: `\lam p_1 … p_n => e : \Pi p_1 … p_n -> E`; untyped lambda params are inferred.

Beta: `(\lam x => b) a` reduces to `b[a/x]`. Eta: `\lam x => f x` is equivalent to `f` when `x`
is not free in `f`.

### Pattern matching in lambda parameters

```ard
\lam (p) => e     -- equivalent to: \lam x => \let (p) => x \in e
```

### Implicit lambda (`__`)

`__` introduces an implicit lambda around the closest enclosing expression into which "a lambda
can be inserted." Lambda insertion is blocked in: case arguments, anonymous class extensions,
right after `\new`, on the left of a function application, under projections, under `\Pi`/`\Sigma`,
and under binary operators.

```ard
\case f __ \with { … }        == \lam x => \case f x \with { … }
f __.1                        == f (\lam p => p.1)
__ a + __ `f` g __ (h __ 0)   == \lam x y z => x a + y `f` g z (\lam t => h t 0)
+ (f __) __                   == \lam x => + (\lam y => f y) x
__ -> \Sigma __ __            == \lam A B C => A -> \Sigma B C
```

---

## Sigma Types

Source: `language-reference/expressions/sigma`.

```ard
\Sigma p_1 … p_n    -- dependent tuple type
```

If `A_i : \Type p_i h_i`, then `\Sigma p_1…p_n : \Type p_max h_max`.

```ard
\Sigma p_1 … p_n (x_1 … x_k : A) q_1 … q_m
-- equivalent to:
\Sigma p_1 … p_n (x_1 : A) … (x_k : A) q_1 … q_m
```

Tuple construction/projection:

```ard
(a_1, … a_n)     -- : \Sigma (x_1 : A_1) … (x_n : A_n), when a_i : A_i[a_1/x_1,…]
p.i              -- projection, i-th field
```

`(a_1,…,a_n).i` reduces to `a_i`. Eta: `(p.1,…,p.n)` is equivalent to `p`.

The typechecker often can't infer a dependent Sigma type for a bare tuple literal and guesses
non-dependent by default; disambiguate with an explicit type ascription:

```ard
((a_1, … a_n) : \Sigma (x_1 : A_1) … (x_n : A_n))

-- or, only for NON-dependent Sigma types, per-field type annotations:
(b_1 : B_1, … b_n : B_n)
```

### Property fields in Sigma

```ard
\Sigma p_1 … (\property p_i) … p_n
```

Works like record `\property`: the field's type must live in `\Prop`, and `(a_1,…,a_n).i` does
**not** evaluate for a property field `i`.

---

## Let and Have

Source: `language-reference/expressions/let`.

```ard
\let | p_1 => e_1
     ...
     | p_n => e_n
\in e_{n+1}
```

Patterns are variables or tuples `(p_1', … p_k')` of patterns (matched against a `k`-fold Sigma
type or `k`-field record, using eta so the concrete shape of `e_i` doesn't matter):

```ard
\let (x,y) => z
\in e     -- evaluates to e[z.1/x, z.2/y] (or field-access form for records)
```

Explicit type for a bound var: `| p_i : E_i => e_i`. Lambda-style extra params on a variable
binding:

```ard
| x_i p^i_1 … p^i_{n_i} => e_i
-- equivalent to:
| x_i => \lam p^i_1 … p^i_{n_i} => e_i
```

### Pattern matching on constructors

```ard
\let (con x_1 ... x_k) => e1
\in e2

-- equivalent to:
\case e1 \with {
  | con x_1 ... x_k => e2
}
```

### `\let!` (strict let)

Same syntax, but `e_1…e_n` are evaluated immediately (performance only).

### `\have`

Same syntax as `\let`, but bound variables **do not unfold** in the body — useful for proof terms
whose actual value shouldn't matter:

```ard
\have x => 0
\in (idp : x = 0)     -- FAILS: x does not unfold to 0 under \have
```

`\have!` is the strict variant of `\have` (mentioned as existing; no example given in the docs).

---

## Case Expressions

Source: `language-reference/expressions/case`.

```ard
\case e_1, ... e_n \with {
  | p_1^1, ... p_n^1 => d_1
  ...
  | p_1^k, ... p_n^k => d_k
}
```

Reduces exactly like a pattern-matching `\func` body (see Functions). If the type can't be
inferred, annotate with `\return`:

```ard
\case e_1, ... e_n \return T \with { ... }
```

Full general form, binding each scrutinee to a name/type usable in later scrutinees and in `T`:

```ard
\case e_1 \as x_1 : E_1, ... e_n \as x_n : E_n \return T \with { ... }
```

(`\as x_i` and `: E_i` are each optional.)

### `\scase`

Related to `\case` as `\sfunc` is to `\func` — never evaluates on its own:

```ard
\scase e_1, … e_n \with { … }
\peval \scase e_1, … e_n \with { … }   -- forces/proves evaluation, as with \sfunc/\peval
```

### `\elim` in `\case`

When a scrutinee is a variable, `\elim` in front substitutes it through the types of later
scrutinees and the result type:

```ard
\case \elim x, e \with {
  | pat, pat' => result
}
-- equivalent to:
\case x \as x, e : E \return A \with {
  | pat, pat' => result
}
```

---

## Class Extensions and `\new`

Source: `language-reference/expressions/class-ext`.

A **class extension** is `C { | f_1 => e_1 … | f_n => e_n }` for record `C` with fields
`f_1…f_n`. Equivalently, `C e_1 … e_n` positionally fills the *not-yet-implemented* fields in
declaration order. `C {}` is equivalent to plain `C`. `C { I }` (a partial extension with field
subset `I` implemented) is a subtype of `C { I' }` iff `I' subset I`.

`\new C { I }` requires **all** fields to be implemented for it to be a value, though the
typechecker can infer missing implementations from the expected type:

```ard
\record R (x y : Nat)
\func f (r : R 0) => r.y
\func g => f (\new R { | y => 1 })   -- x inferred to be 0 from f's expected arg type
```

Updating an existing instance:

```ard
-- if c : C with fields f_1..f_n:
\new c   -- == \new C { | f_1 => c.f_1 ... | f_n => c.f_n }

-- partial field override:
\new c { | f_{i_1} => e_1 ... | f_{i_k} => e_k }
```

---

## `\box` and `\property` erasure

Source: `language-reference/expressions/box`.

If `A : \Prop` and `a : A`, then `\box a : A` does not evaluate to anything but is judgmentally
equal to *any other* boxed element of `A` — this includes every `\box b` and every lemma call.

A parameter whose type is a proposition can be marked `\property`; every call-site argument for
it is then automatically wrapped in `\box`.

---

## Goals (`{?}`)

Source: `language-reference/expressions/goals`.

```ard
\func f (x y : Nat) (p : x = y) : y = x
  => {?}
```

produces:

```
[GOAL] test.ard:1:44:
  Expected type: y = {Nat} x
  Context:
    y : Nat
    x : Nat
    p : x = {Nat} y
  In: {?}
  While processing: f
```

`{?id}` names the goal (name appears only in diagnostics, has no semantic effect).

---

## Prelude

Source: `language-reference/prelude`. Prelude is implicitly imported into every module (can be
imported explicitly to hide/rename). Note per the docs: most of the following are only
*approximated* by ordinary syntax — the real definitions are compiler primitives that can't be
written in plain Arend.

### Nat and Int

`Nat`, `Int`, `Nat.*`, `Nat.<=`, `Nat.div` are genuinely ordinary (if more efficient) definitions.
`Nat.divMod`, `Nat.divModProp`, `Nat.modProp` are omitted from the docs but are themselves
definable in Arend. `Nat.+`/`Nat.-` have *extra* built-in computation rules beyond what an
ordinary pattern match could express:

```
n + 0 => n
n + suc m => suc (n + m)
0 + m => m
suc n + m => suc (n + m)

0 - m => neg m
n - 0 => pos n
suc n - suc m => n - m
```

`Nat.mod (suc n) : Fin (suc n)`; `Nat.divMod (suc n) : \Sigma Nat (Fin (suc n))`.

### Fin

`Fin n` is a subtype of `Nat` and also a subtype of `Fin (suc n)`. `Nat`'s constructors double as
`Fin` constructors: `zero : Fin (suc n)`; if `x : Fin n` then `suc x : Fin (suc n)`.

### Array / DArray

`Array` is a record of an element type `A`, a length `len`, and a function `Fin len -> A`.
`Array A` behaves as the type of lists of `A`; `Array A n` as vectors of length `n`. Constructors
`nil` and `::` are available for pattern matching, so arrays double as both lists and functions
`Fin len -> A`. `DArray` generalizes `Array` to a **dependent** family `A : Fin len -> \Type`,
with elements given by `\Pi (j : Fin len) -> A j`.

### Interval `I` and squeeze functions

```ard
\data I | left | right
```

This *looks* like a plain 2-element type, but isn't: conceptually there is an extra (inaccessible)
constructor connecting `left` and `right`, so **ordinary pattern matching on `I` is forbidden**.
Functions out of `I` are built instead from the primitives `coe`, `coe2`, `squeeze`, `squeezeR`.

```
squeeze left j => left
squeeze right j => j
squeeze i left => left
squeeze i right => i

squeezeR left j => j
squeezeR right j => right
squeezeR i left => i
squeezeR i right => right
```

(These *could* be defined in terms of `coe`, but are primitives for efficiency.)

### Path and `=`

`Path A a a'` is conceptually all functions `\Pi (i : I) -> A i` that additionally satisfy the
computational endpoint conditions `f left => a` and `f right => a'`. The typechecker checks these
conditions when you write `path f`:

```ard
\func test : 1 = 0 => path (\lam _ => 0)
```

produces:

```
[ERROR] test.ard:1:23: The left path endpoint mismatch
  Expected: 1
    Actual: 0
  In: path (\lam _ => 0)
  While processing: test
```

Homotopy level of `Path A a a'`: if `A : (n+1)-type` (`-1 <= n`), `Path A a a'` is an `n`-type;
otherwise it has `A`'s own homotopy level.

`=` is Prelude's genuinely-correctly-defined infix form of `Path`. `@` (path application) is also
correctly defined, but the typechecker gives it a special eta rule not available to user-defined
`@`-named functions elsewhere:

```ard
path (\lam i => p @ i) == p
```

There is also an implicit coercion both ways between `Path A a a'` and `\Pi (i : I) -> A i`: a
bare function of that Pi-type is auto-wrapped in `path` where a `Path`/`=` is expected, and a path
is auto-converted to a function via `@`.

### `Path.inProp`

Has no body — it *postulates* proof irrelevance for `\Prop`: any two elements of a `\Prop`-typed
type are equal. Signature (used above): `Path.inProp : \Pi {A : \Prop} (a a' : A) -> a = a'`.

### `idp`

`idp` is the reflexivity constructor of `Path`/`=`; it is not a "correct" definition since
constructors normally can't use lambdas, but it can replace the `J` eliminator via pattern
matching:

```ard
\func J {A : \Type} {a : A} (B : \Pi (a' : A) -> a = a' -> \Type) (b : B a idp) {a' : A} (p : a = a') : B a' p \elim p
  | idp => b
```

After matching `p` with `idp`, `a` and `a'` become interchangeable (`a'` reduces to `a`).
Restriction: to match `p : e = e'` with `idp`, at least one of `e`/`e'` must be a variable not
occurring in the other, and that variable's substitution must be well-scoped at every occurrence.

Also usable inside `\case`:

```ard
\func J {A : \Type} {a : A} (B : \Pi (a' : A) -> a = a' -> \Type) (b : B a idp) {a' : A} (p : a = a') : B a' p
  => \case \elim a', \elim p \with {
    | _, idp => b
  }
```

### `coe` and `coe2`

`coe` is the eliminator for `I`: given a type family over the interval, it transports an element
from the fiber over `left` to the fiber over an arbitrary point (used to prove `I` contractible
and that `=` satisfies ordinary identity-type rules). Not a "correct" definition (uses pattern
matching on `I` internally), but has one extra reduction rule:

```
coe (\lam x => A) a i => a    -- when x is not free in A
```

`coe2` generalizes `coe` to transport between *any two* fibers over the interval (not just from
`left`).

### `iso`

Not a "correct" definition (also relies on interval pattern matching) — implies the univalence
axiom. No field/signature breakdown given on the page beyond this.

---

## Termination Checker

Source: `language-reference/term-checker`.

Non-terminating definitions would make the type theory inconsistent under Curry–Howard (e.g.
`\func foo : Nat => suc foo` would let you derive `False`). The checker accepts a recursive
group if **either** of two sufficient criteria holds.

### 1. Circular-dependency detection

Build a call graph (vertices = functions/theorems, edges = calls — two calls to the same
function from the same body are two separate edges) and analyze each strongly-connected
component (SCC) independently. A simple self-recursive function (no mutual recursion) is a
single-vertex SCC with one loop per distinct recursive call.

### 2. Call matrices

Comparison results live in `R = {"?", "=", "<"}` ("no info" / "same" / "strictly smaller"). For a
call `g z_1 … z_s` inside `f`'s body (params `x_1…x_n`), build an `n x m` matrix `C` (rows =
`f`'s params, columns = `g`'s params):

- If `g`'s `j`-th param isn't supplied, `C_ij = "?"`.
- If `f`'s `i`-th param's current elimination-pattern `P_i` is just a variable, `C_ij = "="` iff
  `z_j` is literally that variable, else `"?"`.
- If `P_i = con q_1…q_k` (a constructor pattern):
  - **Exact match**: if `z_j` is a call to the same constructor `con` with all subpattern
    positions comparing `"="`, then `C_ij = "="`; if any position compares `"<"`, `C_ij = "<"`;
    if any is `"?"` or arity is short, `C_ij = "?"`.
  - **Subpattern match**: strip `z_j` down through applications/projections/path-`@` to a plain
    constructor call, replaying the same eliminators backward on each subpattern's type; if the
    result type is a recursive occurrence of the same data type and the stripped expression
    compares `"="` or `"<"` against that subpattern, conclude `C_ij = "<"`. If no subpattern
    matches this way, `C_ij = "?"`.
- **Arrays**: in an array `::`-pattern, the *length* component is excluded from structural-descent
  consideration; only the element and tail count.
- **Limitation**: elimination is only tracked in top-level `\elim`/`\with` clauses of a
  function/theorem — eliminations inside `\case` (even `\case \elim x`) are invisible to the
  checker, so a variable eliminated inside a `\case` and then used in a recursive call yields
  `"?"` for that position.
- **Tuples/records/classes**: if a Sigma/record/class-typed argument at the call site is a tuple
  literal or `\new` expression, its components are compared individually rather than opaquely.

### 3. Criterion 1 — termination order (single-vertex SCCs only)

`f(x_1,…,x_n)` with `m` self-calls (each an `n x n` matrix) admits a termination order if there's
a parameter-index sequence `i_1,…,i_s` such that: every call's diagonal entry at `i_1` is `"<"`
or `"="`; among calls with `"="` there, the diagonal at `i_2` must be `"<"`/`"="`; and so on,
until the final selected index is `"<"` for all remaining calls. Equivalently, the diagonals
(one row per call) can be column-reordered into a staircase:

```
<   *   *   *
<   *   *   *
=   <   *   *
=   <   *   *
=   =   <   *
    .....
```

If satisfied, the checker accepts the SCC outright; mutually-recursive (multi-vertex) SCCs skip
straight to Criterion 2.

### 4. Call-graph completion (feeds Criterion 2)

`R` forms a semiring: `+` = best-evidence union, `*` = relation composition
(`< * < = <`, `< * = = <`, `=` is an identity for the other operand, anything `* ?` or `? * anything = ?`).
Call matrices compose along graph edges (matrix multiplication using these `+`/`*`); the
*completion* computes all such compositions to capture the net effect of full cycles, not just
single edges (a "<" on every single edge does NOT guarantee termination if it cancels out around
a cycle). Completion is capped: if >=100 distinct loop matrices accumulate at one vertex, the
checker gives up and rejects the SCC as (possibly) non-terminating — noted as a real limitation
for e.g. a 9-argument function combining a 9-cycle permutation and an adjacent-swap self-call
(these generate all of `S_9`).

### 5. Criterion 2 — size-change principle

For each SCC vertex, look at every *idempotent* matrix labeling a loop in the *completed* graph.
The SCC is accepted if every such idempotent matrix has a `"<"` somewhere on its diagonal
(intuition: an infinite call sequence eventually produces an idempotent loop matrix by
pigeonhole; a guaranteed `"<"` there means some parameter position strictly decreases infinitely
often, which is impossible for well-founded types).

### Practical takeaways for writing recursive `.ard` functions

- Prefer single-function structural recursion where each recursive call's argument is a strict
  subpattern (e.g. matched via `\elim`/`\with` at the definition's top level) of the
  corresponding parameter — this is the cheapest path to passing (Criterion 1).
  ```ard
  -- Fine: n in the recursive call is a subpattern (predecessor) of suc n
  \func half (n : Nat) : Nat
    | 0 => 0
    | suc 0 => 0
    | suc (suc n) => suc (half n)
  ```
- Do **not** rely on eliminating a variable inside a nested `\case` and then recursing on a
  pattern-derived piece of it — the checker won't see the structural descent there (`"?"` always
  results); eliminate at the function's own top-level `\elim`/`\with` instead.
- Mutual recursion is allowed (functions calling each other) as long as there's a linear
  declaration order for signatures and the whole call graph passes Criterion 1 (if collapsible to
  one vertex) or Criterion 2 via the completed call graph.