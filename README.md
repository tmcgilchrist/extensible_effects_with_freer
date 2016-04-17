# Freer Monads, More Extensible Effects

1. Outline
  What am I talking about? Paper by Oleg in Freer Monads.
  May contain some review of the Free monad, no excessive support for the whole free monad needed.

2. Acknowledge the issue being addressed, composability of Effects ie Monads.
  There are 3 approaches proposed for addressing this:
  1. Monad Transformers aka MTL
  2. combine monads via a co-product (Data types ala carte)
  3. effects as an interaction and having effect handlers

3. Represent Effects as data types.

Having a look at Reader we get

``` haskell
data It i a = Pure a
            | Get (i -> It i a)
```

Similarly for Writer and State, we can derive a data type for each

Not extensible, if we want to create something combining Reader / Writer.

4. Free Monad


The pattern of pure and effectful parser and co-variant recursive occurances can be captured as:

``` haskell
data Free f a = Pure a
              | Impure (f (Free f a))
```
where f is a Functor

The concrete instances of f define the particular effectful computations.

Single instance of `Free` sufficies for all, show how to combine Reader/Writer using Free.

Monad's don't compose in general, but this form of Monad will compose because it's defined via a functor.
And functors do compose.


5. Freer Monad

Freer in that it does what free does but is without the Functor constraint.
Lets see how that's possible.

The purpose of the `fmap` is to extend the computation via (>>>) Keisli Composition.

Since this extending is handled in a common way, we could pull out of the request signature and place it
into the fixed request data structure as the second argument of `Impure`

``` haskell
data FFree f a where
    Pure :: a -> FFree f a
    Impure :: f x -> (x -> FFree f a) -> FFree f a
```

Functor implementation is supplied externally.

This yields some performence improvements:
 1. Direct access to the continuation means no rebuilding the mapped data like with fmap
 2. Can change the representation of the continuation to be more efficient. More on this later

6. Extensible Effects

Define what extensible is:

`A Monad type is extensible if we can add  a new effect without having to touch or even recompile the old code.`

MTL doesn't provide this, describe why. I suppose it's related to adding a new effect/monad requires changing code / signatures all over the place.

Modeling the Effects as a data type is not extensible. A data type has a fixed number of variants, adding a new one forces code changes. Basically a data type is a closed union.

What we need is an open union like the following abstract type

`Union (r :: * -> *)`

The first argument r is a type level list of effect labels.
The second argument is the response type.

A concrete union has a single effect label and response type.

Next we'd like to talk about effects in isolation, the `freer` library provides this typeclass.

``` haskell
class Member t r where
  inj :: t v -> Union r v
  prj :: Union r v -> Maybe (t v)
```

This allows us to assert a label `t` occurs in the list `r`.

``` haskell
data FEFree r a where
  Pure :: a -> FEFree r a
  Impure :: Union r x -> (x -> FEFree r a) -> FEFree r a
```
Pure is fairly straight forward.

Impure takes a concrete union of a single effect and response type, a function from `x` to `FEFree r a` and gives you that.

7. Code
**** Switch to Showing Implmementation of Reader and Writer *****

- Type inferencing isn't great when using `ask`


8. Performance issues.

The original `Free` monad while elegant had poor performance.

``` haskell
(((return >>> addGet) >>> addGet) >> addGet) 0


((Impure (inj Get) return . (+0)) >>= addGet) >>= addGet

Impure (inj Get) ((return . (+0) >>> addGet x) >>> addGet)

```

* bind traverses the left argument but passes around the right argument
* performs poorly on left-associated list appends
* [31] "Reflections without remorse: Revealing a hidden sequence to speed up monadic reflection" by Oleg

9. Final Result: Freer and Better Extensible Eff Monad.

``` haskell
data FEFree r a where
  Pure :: a -> FEFree r a
  Impure :: Union r x -> (x -> FEFree r a) -> FEFree r a
```

* with request continuation exposed, represent it in other ways

``` haskell
instance Monad (FEFree f) where ...
  Impure fx k' >>= k = Impure fx (k' >>> k)
```

* represent the continuations as a concrete sequence [31] "Reflections without remorse..."

``` haskell
type Arr r a b = a -> Eff r b
```
* type appreviation for the request continuation.

``` haskell
type FTCQueue (m : * -> *) a b ...

tsingleton :: (a -> m b) -> FTCQueue m a b
(|>) :: FTCQueue m a x -> (x -> m b) -> FTCQueue m a b
```

* type aligned sequence minimal interface `Data.FTCQueue`
* enforce the variant, the result type of one function matches the input of the next one.

10. Examples

Show examples in Emacs

11. Performance Evaluation

11.1 Monad Stacks
 * For deep monad stacks, EE runs in constant time, while MTL takes linear time in the number of layers. Also 40% faster than previous EE based off Free
 * Cost of deep MTL stacks is severe. Implications for design.

11.2 Memory Usage
 * Memory usage for EE is linear with the number of layers. MTL is quadratic in the number of layers. EE is more memory efficient than MTL

11.3 Single effect
 * State benchmark shows MTL is 30 times faster than EE. Special optimisation passes for MTL State :-)
 * Non optimised version the performance between EE and MTL is comparible
