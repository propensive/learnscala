## The Monad Abstraction

The abstraction of a _monad_ is one of the _most useful_ we will encounter when writing Scala, at the same time
as being one of the _most difficult_ to relate to! The challenge of understanding what monads are arises from
their highly abstract nature, and the fact that many of the concrete examples of monads we will learn can seem
so different from each other that it's very hard to see the essence of what makes them _monadic_.

## Functor

Before we approach the definition for a monad, we first ought to look at a simpler, more common abstraction
called a _functor_—which closely relates to, but should not be confused with a _function_.

A functor provides a way to transform a type with a generic type parameter into the same type with a _different_
generic type parameter, by means of a function from the _source_ type parameter to the _destination_ type
parameter.

In Scala, this transformation is done with a method called `map`, and is commonly called a _mapping_.

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
but it's easier for now to consider only a `map` method defined on the type itself. Here is the signature of the
`map` method defined within the type `List[A]`, where the generic parameter `A` is abstract:
```scala
trait List[A]:
   def map[B](fn: A => B): List[B]
```

That definition of `map` will look the same for any generic type, except that the type `List` will be replaced
with whichever type the functor is for.

Given this signature, we are limited in the possible ways it could be implemented. Firstly, we would need to
construct a new `List[B]`, which must either be empty, or contain at least one element of the abstract type `B`.
And without knowing `B` concretely, our only way to construct an instance of `B` is using the function `fn`.

But `fn` must be applied to an instance of type `A`, which is also abstract, though a non-empty `List[A]` also
has access, through its `head` method, for example, to an instance of `A`, and potentially more in its `tail`.

So an implementation of `map` for a `List` could look similar to this:
```scala
trait List[A]:
   def map[B](fn: A => B): List[B] =
      if isEmpty then Nil
      else fn(head) :: tail.map(fn)
```

## Mapping and Flattening

A generic type having a functor makes it possible to imagine what would happen if we were to map using a
function from the generic type parameter to a new instance of the generic type itself (with potentially a
different type parameter).

For example, we could define the function,
```scala
def triplicate[A](a: A): List[A] = List(a, a, a)
```
that converts any input into a list containing that same value three times, and we could apply that to a
`List[Int]`, for example,
```scala
val xs = List(1, 2, 3).map(triplicate)
```
to produce a result of `List(List(1, 1, 1), List(2, 2, 2), List(3, 3, 3))`, which is a `List[List[Int]]`.

Furthermore, there is an obvious way to collapse or "flatten" this `List[List[Int]]` into a `List[Int]` by
simply joining all the inner list elements together. Hence, we could flatten
`List(List(1, 1, 1), List(2, 2, 2), List(3, 3, 3))` into `List(1, 1, 1, 2, 2, 2, 3, 3, 3)`.

## The `flatMap` method

The _flatten_ operation takes no parameters, so the combined operation of mapping and then flattening could, in
fact, be performed as a single step, and in Scala this is implemented with a method called `flatMap`. For a
`List[A]`, `flatMap` looks like this, shown alongside `map` for comparison:
```scala
trait List[A]:
   def map[B](fn: A => B): List[B]
   def flatMap[B](fn: A => List[B]): List[B]
```

The shapes of `map` and `flatMap` are very similar—both are defined on `List[A]`, both introduce a new type
parameter `B`, both take a function from `A`, and both return a `List[B]`. The only difference is that the range
of the function parameter to `flatMap` is `List[B]` instead of `B`.

In Scala, a type which has `map` and `flatMap` methods with these signatures is _monadic_. That is, the type has
a corresponding _monad_, and by this definition, the `List` type has a monad.

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
map first to an `Option[Option[B]]` before flattening it to an `Option[B]`. The given implementation will
produce the same answer as the hypothetical two-step version, but is both simpler and more efficient, as it
avoids unnecessary object construction for the intermediate value.

## The Essence of Monadicity

`List` and `Option` are just two examples of types with monads, but what all monads have in common is that they
provide meaning or purpose to the operation of flattening a doubly-nested generic type, that is, a generic type
parameterized with a different incarnation of the same generic type.

This could be a `List[List[String]]`, an `Option[Option[Int]]`, or a `Try[Try[Date]]`. In each of these
examples, the double-nesting represents some structure that can be "elided", or "simplified", or "fused",
depending on how we choose to mentally model that operation.

For example, an `Option[Option[Boolean]]` has four possible values,
 - `None`,
 - `Some(None)`,
 - `Some(Some(true))`, and
 - `Some(Some(false))`
but the monadic nature of `Option` allows us lose any distinction between `None` and `Some(None)`, to leave just
three possible values of `Option[Boolean]`:
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

However, a successful failure is still a failure, and only a successful success provides us with a value, which
could be considered truly successful, so we can flatten the `Try[Try[T]]` into either a `Success` containing the
value of type `T`, or a `Failure` containing an exception.

## A summary

All these examples have a few things in common:
- they are generic types,
- these generic types have corresponding functor implementations, which means that a function whose range is the
  same as the generic type can be applied to produce an instance of a doubly-nested incarnation of the generic
  type,
- there is a meaningful way to flatten the resultant doubly-nested type

This definition, imprecise as it is, nevertheless broadly describes all monads, and may be the best starting
point for developing an intuition for what a monad is. The full definition requires certain other properties in
the definitions of `map` and `flatMap`, without which certain useful invariant properties of monadic types would
not hold.

But we will study these details later; for now they are less important than understanding the core nature of a
monad.

Having identified this common pattern in certain generic types, Scala provides some ways to exploit it. Most
notable of these is the for-comprehension, which provides special first-class syntax for working more
effectively with monadic types.

It can be tempting to think of monads as simply collection types. It's true that most collection types are
monadic, this doesn't encapsulate the full generality of the monad abstraction.

But it can serve as a good starting point. When pondering a question about monads, we can do worse than to try
to imagine a specialized example of the same question using `List` or `Option`, as these are familiar and
easy-to-understand monadic types, and can help to develop that intuition before we attempt to generalize.

It may also be helpful to read other documentation about monads. Many tutorials exist, and they are well known
for drawing a variety of different analogies between monads and real-world concepts. They are usually written
for Haskell or Scala, it should be kept in mind that reading a Haskell-targeted tutorial may require additional
understanding of Haskell syntax.






## Functor laws

The implementation of `map` for `List` is, by almost any assessment, transforms the list in the "obvious" way.
But it's not unique: we could define a different `map` method which returns a `List[String]` with at most one
element (discarding the others), or one which reverses the order of the elements.

While clearly these would be less useful, they would still be implementations which conform to `map`'s
signature. However, they would not be valid as functor implementations, because a functor must also conform to
certain laws which cannot be represented in Scala's type system.

If `X[T]` is a generic type, the two laws required for its `map` method to provide a functor can be stated as
follows—but we should be aware that they may seem too abstract to grasp in this form, so after stating them, we
will take a more simplistic view of what they mean:

1. if `f` is an isomorphic function from `T` to `S`, then `x => x.map(f)` should also be an isomorphism between
   `X[T]` and `X[S]`.
2. for any two functions, `f` and `g`, `x.map(f).map(g)` should produce the same result as
   `x.map { v => g(f(v)) }` for all `x` in `X[T]`.

The first law is defined in terms of an _isomorphic function_ or _isomorphism_ which means that for a function
`f`, from a type `T` to a type `S`, will transform every instance of `T` into a different instance of `S`, and
that for every instance of `S`, there's a corresponding instance of `T` which `f` will map to it (even if it's
not efficient to find that instance of `T`)! This is commonly known as a _one-to-one_ relationship betweer `T`
and `S`.

What this means in practise is that applying an isomorphic function to `map` must preserve differences in input
values as differences in output values (sometimes called _structure-preserving_), and that it must be
theoretically possible (but perhaps not efficient) to recover the input value which the isomorphic function was
mapped onto knowing only the output value.

Isomorphic functions are not that common in Scala, particularly between different types. And the first law says
nothing about the behaviour when mapping a fuction which is not isomorphic.

A trivial isomorphic function is the identity function, `identity`, which maps every value to itself, and it can
provide an easy way to prove that a particular `map` implementation is not a functor.




Scala does not help to enforce these laws in any way, because the properties cannot be enforced with the type
system.


While the `map` method is part of the definition of a functor, it must also be able to accept any function as a
parameter, which  the source parameter type to
another type (which is, by definition, the destination parameter type). So there is nothing unique about the
function `x => x.toString`, and we could use any other functions instead. For example, this,
```scala
def ordinal(n: Int): String = n match
   case 1              => "first"
   case 2              => "second"
   case 3              => "third"
   case n if n%10 == 1 => s"${n}st"
   case n if n%10 == 2 => s"${n}nd"
   case n if n%10 == 3 => s"${n}rd"
   case _              => s"${n}th"

List(3, 6, 9).map(ordinal(_))
```
would produce the list, `List("third", "6th", "9th")`, or this,
```scala
List(3, 6, 9).map("Hello World"(_))
```
which produces the list, `List('l', 'W', 'l')`. This is a `List[Char]` which reflects the type of the applied
function, `"Hello World"(_)`, which maps each integer element, *n*, of the initial list to the *n*th character
of the string `"Hello World"`.

These are examples which _use_ a functor, but the concrete types `Int`, `String` and `Char`, like each function
that is applied with `map` in the examples above, can be _anything_, and are completely unrelated to the functor
itself, which dependends only on `List`.

The functor is defined entirely in the `map` method of `List`. For a `List[A]`, its signature looks like this:
```scala
def map[B](fn: A => B): List[B]
```

This signature limits the ways in which this method can be implemented. Both `A` and `B` are abstract at the
definition site, which means that no concrete properties about instances of `A` or `B` are known.


For example, the `List` type has a monad. We can also say that `List` is a _monadic type_.

Monads concern the sequencing of computations.


Firstly, a monadic type must be generic.

It is useful to first clarify some nomenclature. A monad is a particular set of rules which can apply to
instances of a particular type. We will say that that type _has a monad_, or sometimes, _has a monad instance_.
It is very common to hear that the particular type _is a monad_ or that instances of that type _are monads_, but
we are not going to adopt that convention.

Instead, if we can define the set of monad rules corresponding to a particular type, thne we will say that the
type _is monadic_.


For comprehensions

Scala has first-class syntax for working with monads, in the form of a _for-comprehension_. These rely primarily
on two methods, `map` and `flatMap`, being present on 