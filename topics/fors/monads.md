## The Monad Abstraction

The abstraction of a _monad_ is one of the _most useful_ we will encounter when writing Scala, at the same time
as being one of the _most difficult_ to relate to! The challenge of understanding what monads are arises from
their highly abstract nature, and the fact that many of the concrete examples of monads we will learn can seem
so different from each other that it's very hard to see the essence of what makes them _monadic_.

## Functor

Before we try to understand what a monad is, we first ought to look at a simpler, more common abstraction called
_functor_. Despite having a similar name, a functor is not the same thing as a _function_, though as we will
see, they do have similarities.

A functor provides a way to transform a type with a generic type parameter into the same type with a _different_
generic type parameter, by means of a function from the _source_ type parameter to the _destination_ type
parameter.

In Scala, this transformation is done with a method called `map`, and is commonly called a _mapping_. We often
use the terminology of _mapping a function over_ or _across_ a value, which means applying the function to its
`map` method.

For example, `List` provides a functor, which means that we could transform a `List[Int]` into a `List[String]`
by calling its `map` method with a function from `Int` to `String`, such as `x => x.toString` (which just
converts each number to its canonical string representation).

The resultant `List[String]` has the same number of elements, but now each one is a `String` instead of an `Int`
because the function `x => x.toString` was applied to each one in turn, hence,
```scala
val listOfStrings: List[String] =
   List(3, 6, 9).map { x => x.toString }
```
produces the list, `List("3", "6", "9")`.

## The `map` method

A functor can exist independently of the type it operates on (e.g. `List`), and is often defined as a typeclass,
but it's easier for now to consider a `map` method defined as part of the type itself. The loose relationship
between a type and its functor leads us to say that a type _has a functor_, instead of saying that the type
_is a functor_. But it is still common to hear both used, and they mean the same thing.

Here is the signature of the `map` method defined within the type `List[A]`, where the generic parameter `A` is
abstract:
```scala
trait List[A]:
   def map[B](fn: A => B): List[B]
```

That definition of `map` will look the same for any generic type, except that the type `List` will be replaced
with the functor's particular corresponding type.

Given this signature, we are limited in the possible ways it could be implemented, and we can reason about this
as follows: Firstly, any implementation must construct a new `List[B]`, which must either be empty, or contain
at least one element of the abstract type `B`. And without knowing `B` concretely, our only way to construct an
instance of `B` is using the function `fn`.

But `fn` must be applied to an instance of type `A`, which is also abstract, though a non-empty `List[A]` also
has access, through its `head` method, for example, to an instance of `A`, and potentially more in its `tail`.

So an implementation of `map` for a `List` might look similar to this:
```scala
trait List[A]:
   def map[B](fn: A => B): List[B] =
      if isEmpty then Nil
      else fn(head) :: tail.map(fn)
```

Mapping over the empty list changes nothing. A non-empty list can be transformed by applying the function,
`fn`, to its `head` element, and prepending this to the result of recursing on the `tail`.

The resultant `List[B]` will have the same number of elements as the original `List[A]`.

And in general, applying a functor to an instance of a generic type may change the type parameter, but will not
change the runtime structure of the value: a list will keep the same number of elements; a binary tree will have
branches in the same places before and after the transformation; an `Option` will never swap between `None` and
`Some`. This is enforced by some additional constraints on the definition of a functor, but we will look at them
in a later lesson.

## Flattening

If a generic type has a functor, we could try to imagine what would happen if we use it to map a function whose
return type is another incarnation of the _same_ generic type.

For example, we could define the function,
```scala
def triplicate(n: Int): List[Int] = List(n, n, n)
```
that converts any `Int` input into a list containing that same value three times, and then apply it to a
`List[Int]`, for example,
```scala
val xs = List(1, 2, 3).map(triplicate)
```
to produce a result of `List(List(1, 1, 1), List(2, 2, 2), List(3, 3, 3))`, which is a `List[List[Int]]`.

And in general, if the return type of the function we apply to the `map` method is an incarnation of the same
generic type the `map` is called on, the result of `map` will always be a type doubly-nested within a generic
type. We could say that it always fits the shape, `F[F[A]]`.

Now, in the case where the generic type is `List`, there is an obvious way to collapse or "flatten" the value,
from a `List[List[Int]]` into a `List[Int]`, by joining all the inner list elements together in order. Hence,
`List(List(1, 1, 1), List(2, 2, 2), List(3, 3, 3))` would become `List(1, 1, 1, 2, 2, 2, 3, 3, 3)` under the
flattening operation.

A method `flatten` is often available as an extension method on doubly-nested monadic types like
`List[List[T]]`.

## The `flatMap` method

`flatten` requires no parameters, so the combined operation of mapping and then flattening could, in fact, be
performed as a single step, taking just a function as a parameter, and in Scala this is implemented with a
method called `flatMap`.

For a `List[A]`, `flatMap` looks like this, shown alongside `map` for comparison:
```scala
trait List[A]:
   def map[B](fn: A => B): List[B]
   def flatMap[B](fn: A => List[B]): List[B]
```

The shapes of `map` and `flatMap` are very similar—both are defined on `List[A]`, both introduce a new type
parameter `B`, both take a function from `A`, and both return a `List[B]`. The only difference is that the range
of the function parameter to `flatMap` is `List[B]` instead of `B`. So our earlier experiment where we chose to
apply a function returning a `List` to the `map` method would be less of a choice for `flatMap`, and rather
enforced by the method's signature.

In Scala, a type which has both `map` and `flatMap` methods with these signatures is _monadic_. That is, the
type has a corresponding _monad_, and by this definition, `List` has a monad.

There are a couple of other necessary constraints on how `map` and `flatMap` are defined—"the Monad Laws"—which
we will look at in detail in a later lesson, but we can make progress without learning them yet.

`Option` is also monadic, and we can define its `map` and `flatMap` operations very easily, by pattern matching
on each of the cases, like this:
```scala
trait Option[A]:
   def map[B](fn: A => B): Option[B] = this match
      case Some(value) => Some(fn(value))
      case None        => None

   def flatMap[B](fn: A => Option[B]): Option[B] = this match
      case Some(value) => fn(value)
      case None        => None
```

Here we can also see that we can implement `flatMap` for `Option` directly in a single step, without needing to
map first to an `Option[Option[B]]` before flattening it to an `Option[B]`.

The given implementation will produce the same answer as the hypothetical two-step version, but is both simpler
and more efficient, as it avoids unnecessary object construction for the intermediate value. If we think of
`flatMap` as "mapping then flattening", it might seem an inefficient operation, but it's quite common that it
can be implemented without a high cost to performance. (And at this stage, it's better to focus on correctness
over performance!)

## The Essence of Monadicity

`List` and `Option` are just two examples of types with monads, but what all monads have in common is that they
provide meaning or purpose to the operation of flattening a doubly-nested generic type.

This could be a `List[List[String]]`, an `Option[Option[Int]]`, or a `Try[Try[Date]]`. In each of these
examples, the double-nesting represents a detail of the value's structure that the monad "simplifies", "elides",
or "fuses together".

For example, an `Option[Option[Boolean]]` has four possible values,
 - `None`,
 - `Some(None)`,
 - `Some(Some(true))`, and
 - `Some(Some(false))`
but the monadic nature of `Option` lets us choose to throw away any distinction between `None` and `Some(None)`,
to leave just three possible values of `Option[Boolean]`:
- `None`,
- `Some(true)`, and
- `Some(false)`

Likewise, a `List[List[Int]]` is an ordered sequence of ordered sequences of integers, and we can read the
integers of each inner list in order, in the order that they appear in the outer list, thereby producing a
single ordered sequence of integers, or a `List[Int]`.

In doing this flattening transformation, we consciously discard the additional structure as an insignificant
detail, or an implementation detail.

Finally, a `Try[T]` represents a successful value of type `T`, or a failure (with an exception). So a
`Try[Try[T]]` could represent a successful success, a succesful failure, or a failure!

However, a successful failure is still a failure, and only a successful success provides us with a value that
could be considered truly successful, so we can flatten the `Try[Try[T]]` into either a `Success` containing the
value of type `T`, or a `Failure` containing an exception.

## Summary

All these examples have a few things in common:
- they are generic types,
- these generic types have corresponding functor implementations, which means that a function (whose return type
  is an incarnation of the same generic type) can be applied to produce an instance of a doubly-nested
  incarnation of that type,
- there is a meaningful way to flatten the resultant doubly-nested type

This definition, imprecise as it is, nevertheless broadly describes all monads, and may be the best starting
point for developing an intuition for what a monad is. The full definition requires certain other properties in
the definitions of `map` and `flatMap`, without which certain useful invariant properties of monadic types would
not hold.

But we will study these details later; for now they are less important than understanding the core nature of
monads.

Having identified this common pattern in certain generic types, Scala provides some ways to exploit it. Most
notable of these is the for-comprehension, which provides special first-class syntax for working more
effectively with monadic types.

It can also be tempting to think of monads as simply collection types. It's true that most collection types are
monadic, but collections don't really express the full generality of the monad abstraction.

Nonetheless, it can serve as a good starting point. When pondering a question about monads, we can do worse than
try to imagine a specialized example of the same question using `List` or `Option`, as these are familiar and
easy-to-understand monadic types, and can help to develop that intuition before we attempt to generalize.

It can also be helpful to read other documentation about monads. Many tutorials exist, and they are well known
for drawing a variety of different analogies between monads and real-world concepts. They are usually written
for Haskell or Scala, so it should be kept in mind that reading a Haskell-targeted tutorial may require
additional understanding of Haskell syntax, but that in itself can be insightful.
