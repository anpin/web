---
title: "Category theory for programmers: F# quick reference"
description: "Definitions, laws, and F# examples for categories, functors, monads, universal constructions, adjunctions, and Yoneda."
pubDate: 2026-11-16
tags: ["notes", "category-theory", "fsharp"]
---

Notes based on Bartosz Milewski’s [Category Theory for Programmers][milewski].

F# analogues assume pure, total functions, fully defined values, parametricity, and extensional equality. Exceptions, mutation, nulls, and runtime type inspection fall outside this model.

F# lacks native higher-kinded type parameters. Examples therefore specialize to concrete type constructors. Equations are mathematical laws, not executable equality tests.

`C(A, B)` denotes a hom-set, `Set` the category of sets and functions, and `≅` an isomorphism.

## Categories and composition

### Category

Objects, morphisms with specified sources and targets, associative composition, and an identity for each object. For `f : A → B` and `g : B → C`:

```text
g ∘ f : A → C
id_B ∘ f = f
f ∘ id_A = f
(h ∘ g) ∘ f = h ∘ (g ∘ f)
```

Model types as objects and pure, total functions as morphisms. F# writes [composition][functions] as `g << f` or `f >> g`; identity is `id`.

### Isomorphism

`A ≅ B` when morphisms `f : A → B` and `g : B → A` satisfy:

```text
g ∘ f = id_A
f ∘ g = id_B
```

Example: swapping a pair gives `('a * 'b) ≅ ('b * 'a)`.

### Monoid

A set with an associative binary operation and an identity. Equivalently, a category with one object: elements become endomorphisms, and the operation becomes composition.

Represent the operations explicitly:

```fsharp
type Monoid<'a> =
    { Empty: 'a
      Append: 'a -> 'a -> 'a }
```

```text
m.Append m.Empty x = x
m.Append x m.Empty = x
m.Append (m.Append x y) z = m.Append x (m.Append y z)
```

Example: lists under `(@)`, with identity `[]`. The record type does not enforce the laws.

### Duality

The opposite category `Cᵒᵖ` reverses morphisms and composition: `Cᵒᵖ(A, B) = C(B, A)`.

| Construction   | Dual            |
| -------------- | --------------- |
| Initial object | Terminal object |
| Product        | Coproduct       |
| Limit          | Colimit         |
| Monad          | Comonad         |

## Universal constructions

A **universal property** specifies a construction through unique morphisms relating it to all compatible candidates. Solutions are unique up to a unique isomorphism preserving the specified structure.

### Initial and terminal objects

| Object       | Universal property                         | F# analogue                               |
| ------------ | ------------------------------------------ | ----------------------------------------- |
| Initial `0`  | Exactly one morphism `0 → A` for every `A` | Empty type; no direct built-in equivalent |
| Terminal `1` | Exactly one morphism `A → 1` for every `A` | `unit`, with `fun _ -> ()`                |

`System.Void` is not an empty discriminated union with a total elimination function.

### Product

An object `P` with projections `p : P → A` and `q : P → B`. For every `f : X → A` and `g : X → B`, exactly one `h : X → P` satisfies:

```text
p ∘ h = f
q ∘ h = g
```

In F#, `P = 'a * 'b`, with projections `fst` and `snd`:

```fsharp
let pair (f: 'x -> 'a) (g: 'x -> 'b) (x: 'x) =
    f x, g x
```

### Coproduct

An object `S` with injections `i : A → S` and `j : B → S`. For every `f : A → X` and `g : B → X`, exactly one `h : S → X` satisfies:

```text
h ∘ i = f
h ∘ j = g
```

In F#, use [`Choice<'a, 'b>`][choice-docs], with injections `Choice1Of2` and `Choice2Of2`. The unique handler is `either f g`:

```fsharp
let either f g = function
    | Choice1Of2 a -> f a
    | Choice2Of2 b -> g b
```

Both definitions use [existence and uniqueness][products]. Projections or injections alone are insufficient.

### Exponential object

In a category with binary products, `B^A` has an evaluation morphism `eval : B^A × A → B`. For every `h : X × A → B`, there is a unique `curry h : X → B^A` satisfying:

```text
eval ∘ ((curry h) × id_A) = h
```

In `Set`, `B^A` is the set of functions `A → B`. The F# analogue is `'a -> 'b`:

```fsharp
let curry f x a = f (x, a)
let uncurry f (x, a) = f x a
```

### Limits and colimits

[Limits][limits] generalize products. Colimits generalize coproducts. A **diagram** specifies objects and arrows between them.

- A **limit** `L` has compatible maps **to** the diagram’s objects. Every compatible family of maps from another object `X` is obtained through exactly one map `X → L`.
- A **colimit** `Q` has compatible maps **from** the diagram’s objects. Every compatible family of maps into another object `X` is obtained through exactly one map `Q → X`.

For two objects `A` and `B` with no arrows between them:

- The limit is their **product**. Maps `f : X → A` and `g : X → B` determine exactly one map `x ↦ (f x, g x)` into the pair.
- The colimit is their **coproduct**. Maps `f : A → X` and `g : B → X` determine exactly one handler from `Choice<'a, 'b>` to `X`.

**Compatible** means that following the diagram’s arrows agrees with the supplied maps. For `f : A → C` and `g : B → C`, matching pairs satisfy `f(a) = g(b)`. In `Set`, these pairs form the **pullback**, the limit of `A → C ← B`.

Compatible families into a diagram are **cones**. Families out are **cocones**. A limit is a terminal cone. A colimit is an initial cocone.

| Diagram                      | Limit           | Dual diagram                 | Colimit        |
| ---------------------------- | --------------- | ---------------------------- | -------------- |
| Empty                        | Terminal object | Empty                        | Initial object |
| Two objects, only identities | Product         | Two objects, only identities | Coproduct      |
| `A ⇉ B`                      | Equalizer       | `B ⇉ A`                      | Coequalizer    |
| `A → C ← B`                  | Pullback        | `A ← C → B`                  | Pushout        |

## Functors and natural transformations

### Functor and endofunctor

A functor `F : C → D` maps objects and morphisms, preserving sources, targets, identities, and composition:

```text
F(id_A) = id_(F A)
F(g ∘ f) = F(g) ∘ F(f)
```

An **endofunctor** has `C = D`. Examples in the F# model are `option` with [`Option.map`][option-docs], and `list` with `List.map`.

```text
Option.map : ('a -> 'b) -> 'a option -> 'b option

Option.map id = id
Option.map (g << f) = Option.map g << Option.map f
```

### Natural transformation

For functors `F, G : C → D`, a [natural transformation][natural-transformations] `η : F ⇒ G` assigns components `η_A : F A → G A`. For every `f : A → B`:

```text
G(f) ∘ η_A = η_B ∘ F(f)
```

An F# example, polymorphic in the element type:

```fsharp
let optionToList = function
    | None -> []
    | Some x -> [x]
```

```text
List.map f << optionToList = optionToList << Option.map f
```

### Representable functor

A functor `F : C → Set` is [representable][representable] when naturally isomorphic to `C(R, -)` for some object `R`.

```text
F A ≅ C(R, A)
```

The function type `'r -> 'a` is represented by `'r`. The pair functor `'a ↦ 'a * 'a` is represented by `bool`:

```fsharp
let tabulatePair f = f false, f true
let indexPair (x, y) flag = if flag then y else x
```

```text
indexPair << tabulatePair = id
tabulatePair << indexPair = id
```

## Applicatives, monads, and comonads

### Applicative

An applicative functor combines supplied computations without using an earlier result to choose a later computation. For `option`, define:

```fsharp
let pureOption x = Some x
let (<*>) mf mx = Option.bind (fun f -> Option.map f mx) mf
```

```text
// Identity
pureOption id <*> v = v
// Composition
pureOption (<<) <*> u <*> v <*> w = u <*> (v <*> w)
// Homomorphism
pureOption f <*> pureOption x = pureOption (f x)
// Interchange
u <*> pureOption y = pureOption (fun f -> f y) <*> u
// Compatibility with mapping
Option.map f x = pureOption f <*> x
```

`pureOption` and `(<*>)` are local definitions, specialized to `option`.

### Monad

A monad provides `pure` and `bind` operations for a type constructor `M`, subject to identity and associativity laws.

For `option`, `pureOption x = Some x`. [`Option.bind next`][option-docs] calls `next x` for `Some x`, and returns `None` without calling `next` for `None`.

```fsharp
let tryHalf n =
    if n % 2 = 0 then Some (n / 2) else None

// Some 8 |> Option.bind tryHalf |> Option.bind tryHalf  // Some 2
// Some 6 |> Option.bind tryHalf |> Option.bind tryHalf  // None
```

Both steps return an option. Ordinary mapping would introduce a nested option. Bind combines mapping with flattening:

```text
Option.map tryHalf (Some 8)  = Some (Some 4)
Option.bind tryHalf (Some 8) = Some 4
```

Reuse `pureOption` from the applicative section. Define `join` to remove one option layer, and `(>>=)` as bind with the value first:

```fsharp
let join mmx = Option.flatten mmx
let (>>=) mx k = Option.bind k mx
```

```text
mx >>= k = join (Option.map k mx)
join mmx = mmx >>= id

// Left identity: bind after pure is ordinary application.
pureOption a >>= f = f a
// Right identity: binding with pure changes nothing.
mx >>= pureOption = mx
// Associativity: regrouping preserves the computation and step order.
(mx >>= f) >>= g = mx >>= (fun x -> f x >>= g)

// Compatibility with Applicative
mf <*> mx = mf >>= (fun f -> mx >>= (fun x -> pureOption (f x)))
```

In the [categorical definition][monads-categorical], `M` is an endofunctor, `η : Id ⇒ M` supplies `pure`, and `μ : M ∘ M ⇒ M` supplies `join`. For `option`, `M ∘ M` means two layers: `'a option option`.

### Kleisli category

For a monad `M`, objects are unchanged, and morphisms are `A → M B`. For `option`, these have type `'a -> 'b option`. Identity is `pureOption`; composition uses bind:

```fsharp
let (>=>) f g x = f x >>= g
```

The monad laws give the category laws. This operator uses the `option` bind defined above.

### Comonad

The dual of a monad: an endofunctor `W` with extraction `W ⇒ Id` and duplication `W ⇒ W ∘ W`. For a fixed environment type, use pairs `'env * 'a`:

```fsharp
let mapPair f (env, value) = env, f value
let extract (_, value) = value
let duplicate context = fst context, context
let extend f context = fst context, f context
```

```text
extract << extend f = f
extend extract = id
extend f << extend g = extend (f << extend g)

duplicate = extend id
mapPair f = extend (f << extract)
```

## Algebras and recursion

For an endofunctor `F`, an **[F-algebra][algebras]** is a carrier `A` with a morphism `F A → A`. An **F-coalgebra** is a carrier `A` with a morphism `A → F A`.

An algebra consumes one layer; a coalgebra produces one layer. They supply the steps for folds and unfolds.

### Recursion schemes

Let `μF` be the carrier of an initial `F`-algebra, and `νF` the carrier of a terminal `F`-coalgebra, when they exist.

| Scheme       | Given                                  | Result                                        |
| ------------ | -------------------------------------- | --------------------------------------------- |
| Catamorphism | Algebra `F A → A`                      | Unique algebra morphism `μF → A`              |
| Anamorphism  | Coalgebra `A → F A`                    | Unique coalgebra morphism `A → νF`            |
| Hylomorphism | Coalgebra `A → F A`, algebra `F B → B` | Recursive unfold-and-fold computation `A → B` |

Specializing to list layers, finite structures, and terminating computations:

```fsharp
type ListF<'item, 'next> = NilF | ConsF of 'item * 'next
type FixList<'a> = FixList of ListF<'a, FixList<'a>>

let mapListF f = function
    | NilF -> NilF
    | ConsF (item, next) -> ConsF (item, f next)

let rec cata alg (FixList layer) =
    alg (mapListF (cata alg) layer)

and ana coa seed =
    FixList (mapListF (ana coa) (coa seed))

and hylo alg coa seed =
    alg (mapListF (hylo alg coa) (coa seed))
```

For finite unfoldings, `hylo alg coa = cata alg << ana coa`. In general, `μF` and `νF` can differ. This eager `ana` does not produce infinite structures.

### Free monad

A free monad on a functor `F` represents returned values and trees of `F` operations. For `F X = string × X`:

```fsharp
type FreeWrite<'a> =
    | Pure of 'a
    | Write of string * FreeWrite<'a>

let rec bindFree k = function
    | Pure value -> k value
    | Write (message, next) -> Write (message, bindFree k next)
```

`Pure` is the unit; `bindFree` substitutes computations at returned values. A natural transformation `F ⇒ M` extends uniquely to a monad morphism `Free F ⇒ M`.

A concrete interpreter collects messages and returns the final value:

```fsharp
let rec runFree = function
    | Pure value -> [], value
    | Write (message, next) ->
        let messages, value = runFree next
        message :: messages, value
```

## Adjunctions and Yoneda

### Adjunction

Currying gives a reversible correspondence between functions that take a pair and functions that take one argument and return another function:

```text
('x * 'a -> 'b) ≅ ('x -> 'a -> 'b)
```

Using the earlier `curry` and `uncurry` definitions:

```fsharp
let addPair (x, y) = x + y
let addCurried = curry addPair

// addPair (2, 3) = 5
// addCurried 2 3 = 5
// uncurry addCurried (2, 3) = 5
```

The conversions preserve the function’s behavior and are inverses:

```text
uncurry (curry h) = h
curry (uncurry k) = k
```

In `Set`, fix `A` and take `F X = X × A` and `G B = B^A`. These functors are [adjoint][adjunctions]: morphisms `F X → B` correspond bijectively to morphisms `X → G B`.

In general, for `F : C → D` and `G : D → C`, an adjunction `F ⊣ G` is a family of bijections:

```text
D(F X, B) ≅ C(X, G B)
```

The bijections must respect composition in `X` and `B`. This is the naturality condition.

### Yoneda lemma

An optional value can be represented by an operation that accepts any function on its contents. For example, `Some 3` gives the operation `f ↦ Option.map f (Some 3)`.

Encode this operation with a [generic interface method][generics], so the caller can choose any result type `'r`:

```fsharp
type IOptionYoneda<'a> =
    abstract Run<'r> : ('a -> 'r) -> 'r option

let toYoneda (value: 'a option) : IOptionYoneda<'a> =
    { new IOptionYoneda<'a> with
        member _.Run<'r>(f: 'a -> 'r) : 'r option =
            Option.map f value }

let fromYoneda (transform: IOptionYoneda<'a>) : 'a option =
    transform.Run id

let encoded = toYoneda (Some 3)

// encoded.Run ((+) 1) = Some 4
// encoded.Run string = Some "3"
// fromYoneda encoded = Some 3
```

Applying `Run` to `id` recovers the optional value. [Yoneda][yoneda] says that, for a natural operation, this value determines every other result:

```text
transform.Run f = Option.map f (transform.Run id)
```

Thus `toYoneda` and `fromYoneda` preserve all information. This assumes total, parametric implementations of `Run`; the interface alone does not enforce naturality.

In general, for locally small `C`, an object `A`, and `F : C → Set`:

```text
Nat(C(A, -), F) ≅ F A
```

`C(A, -)` sends each object `R` to the set of maps `A → R`. `Nat` denotes natural transformations. The two directions are evaluation at identity and mapping:

```text
value = α_A(id_A)
α_R(f) = F(f)(value)
```

Substituting `F = C(B, -)` gives the covariant hom-functor form of the [Yoneda embedding][yoneda-embedding]:

```text
Nat(C(A, -), C(B, -)) ≅ C(B, A)
```

Given `h : B → A`, the corresponding transformation sends `k : A → R` to `k ∘ h : B → R`. Applying the transformation to `id_A` recovers `h`, explaining the direction `'b -> 'a`.

For the identity functor, the representation accepts a function `A → R` and returns its result. Applying it to `id` recovers the value of `A`:

```text
(∀R. (A → R) → R) ≅ A
```

[milewski]: https://bartoszmilewski.com/2014/10/28/category-theory-for-programmers-the-preface/
[functions]: https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/functions/
[choice-docs]: https://fsharp.github.io/fsharp-core-docs/reference/fsharp-core-fsharpchoice-2.html
[option-docs]: https://fsharp.github.io/fsharp-core-docs/reference/fsharp-core-optionmodule.html
[generics]: https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/generics/
[products]: https://bartoszmilewski.com/2015/01/07/products-and-coproducts/
[limits]: https://bartoszmilewski.com/2015/04/15/limits-and-colimits/
[natural-transformations]: https://bartoszmilewski.com/2015/04/07/natural-transformations/
[representable]: https://bartoszmilewski.com/2015/07/29/representable-functors/
[monads-categorical]: https://bartoszmilewski.com/2016/12/27/monads-categorically/
[algebras]: https://bartoszmilewski.com/2017/02/28/f-algebras/
[adjunctions]: https://bartoszmilewski.com/2016/04/18/adjunctions/
[yoneda]: https://bartoszmilewski.com/2015/09/01/the-yoneda-lemma/
[yoneda-embedding]: https://bartoszmilewski.com/2015/10/28/yoneda-embedding/
