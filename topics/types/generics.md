## Parameterized Types

We should think of a type as a shorthand for a set of properties which are true about all of its instances. For
simple types, like `String`, `Boolean` or `scala.Range`, all of those properties will be defined in terms of
other known and invariant types.

The properties of these types are inferred directly from the concrete method definitions in the templates which
are used to create instances of these types: the object, class or trait definitions that instantiate them.

For example, the method `Range#by` takes an `Int` and returns a `Range`, and `String#capitalize` takes no
parameters and returns a `String`. These are invariant for different instances of `Range` or `String`.

Other types, like `Option[T]`, always take a type parameter, and will have some properties, like
`Option#isEmpty`, which consistently return a `Boolean`, but others, like `Option#get` which return `T`—a type
which can be different for different instances—and which depend directly on the type parameter, `T`. These are
known as _generic types_.

For example, the method `Option[Int]#get` will return an `Int`, and `Option[String]#get` will return a `String`.

Other methods of `Option`, such as `toSet` (which returns the empty set when the `Option` is `None` or a
one-element set when the option is `Some`), return a type which is indirectly dependent on the `Option`'s type
parameter; a `Set[Int]` for an `Option[Int]`, a `Set[Exception]` for an `Option[Exception]`, and in general, a
`Set[T]` for an `Option[T]`.

A value having a generic type reflects the fact it has been instantiated from a generic template, and that
concrete choices have necessarily been made about the template's type parameters.

Usually when we reference a type like `Option[T]` in code, the `[T]` parameter is a mandatory part of the type,
and can't be omitted. There are some exceptions to this rule, but we will not cover them yet. (Nevertheless, in
prose it's sometimes more convenient to talk about the "type" called `Option` without specifying its parameter.)
A consequence of this is that the type must always be fully specified at every usage, and the set of properties
the generic type represents will always be known, in terms of the type parameter specified.

A generic type may have any number of parameters, for example `Either[L, R]` takes two: one which corresponds to
the type of the `Left` branch, and one which corresponds to the `Right` branch.

There is also nothing to stop a generic type parameter being a generic type itself, so `Option[Option[Int]]` is
a valid type which could represent the values, `None`, `Some(None)` or `Some(Some(2))`. Another example might be
`List[Either[Option[Int], String]]`.

## Defining a generic type

When defining a template for a generic type, we can write code which operates on instances of the generic type
parameter. But because we don't know _concretely_ what this type parameter will be for any particular instance
of the generic type, we are not allowed to presume anything about it.

In this context, the type parameter is called an _abstract type_ because it's not known _concretely_.

We could define a template, for example,
```scala
trait Consumer[T]:
   def take(value: T): Unit
```
and any implementation of the method, `take`, could include expressions and statements involving `value`, which
has the type `T`. But the type `T` is abstract in the context of this method definition, which means that for
different instances of the generic type it could be instantiated to `Boolean` or `String` or any other type.

Hence, the set of properties of `T` that can safely be assumed in an implementation of the method `take` is just
those which are common to all values: the set represented by the type `Any`. And even without knowing anything
about `T`, we do at least know that it is a subtype of `Any`.

But despite not knowing any of the properties of `T`, we can still utilize the obvious property that an instance
of `T` is an instance of the same type `T`, which is true whether `T` is `String` or `Long` or anything else for
a particular instance.

This means that we could extend our template, `Consumer[T]`,
```scala
trait Consumer[T]:
   def take(value: T): Unit

   def takeAll(values: List[T]): Unit =
      for value <- values do take(value)
```
and the method `takeAll` can be implemented by passing a instances of `T` (each `value`) to a method which takes
instances of `T` as parameters. And regardless of what concrete type `T` is for a particular instance of
`Consumer[T]`, this is safe.

## Subtyping

The subtyping relationships between different generic types are nontrivial, unfortunately!

`List[Exception]` is a subtype of `List[Throwable]`, which means that a method which takes an instance of
`List[Throwable]` may be passed an instance of `List[Exception]`. This is true, in part, because `Exception` is
a subtype of `Throwable`.

But in the case of `Ordering[T]`, a type which provides a `compare` method which determines whether one instance
of `T` is greater than, less than or equal to another instance of `T`, an `Ordering[Exception]` is _not_ a
subtype of `Ordering[Throwable]`, despite the subtyping relationship between `Exception` and `Throwable`.

This should make some sense with an example: if we define a template with a method for sorting values (without
any implementation for now),
```scala
trait Sorter[T]:
   def sort(values: List[T], ordering: Ordering[T]): List[T]
```
then we could create an instance of `Sorter[Throwable]` whose `sort` method would have the signature,
```scala
def sort(values: List[Throwable], ordering: Ordering[Throwable]): List[Throwable]
```
and this could be called with a list of `Throwable`s and an `Ordering[Throwable]`, and the `compare` method of
the `Ordering` can happily compare any two elements in the `List[Throwable]` because they are instances of
`Throwable`.

The same would be true if we call `Sorter[Throwable]#sort` with a `List[Exception]` (which is a subtype of
`List[Throwable]`) and the same `Ordering[Throwable]`: the `compare` method can compare any two `Throwable`
instances, and that includes `Exception` instances, of course.

But we _could not_ call `Sorter[Throwable]#sort` with a `List[Throwable]` and an `Ordering[Exception]` because
`Ordering[Exception]` is _not_ a subtype of `Ordering[Throwable]`.

And nor would we want to! An `Ordering[Exception]` would not capable of comparing any two `Throwable` instances
in the list. Its implementation may depend on certain properties of those `Exception`s, which are not true for
other `Throwable`s. And for this reason, it would not be correct for `Ordering[Exception]` to be a subtype of
`Ordering[Throwable]`.

The subtyping relationships between generic types are determined by the _variance_ of their type parameters,
which is a complex topic which we will cover in more detail later. For now, it should be sufficient to be aware
that some generic types may assume the same subtyping relationships as their type parameters, and these are
called _covariant_, and others do not.

Like any other type, all generic types fit into the type hierarchy, which will reflect these subtyping
relationships.

