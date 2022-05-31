## Uncertain Generics

Generic types in Scala provide a way of describing a set of values, some of whose properties are dependent on
another type, which is specified as a parameter to the type whenever it is used.

Like all types, each generic type represents a set of properties which are true about its instances. Some of
these properties may depend on the generic type parameter, such as the `toSet` method of `Option`, which will
return a `Set` of the same type as the `Option`, while others are not dependent on the type parameter, like
`Option`'s `isEmpty` method which _always_ returns a `Boolean`.

A _wildcard type_ is a type which can represent those invariant properties precisely, while retaining just an
imprecise approximation of those dependent properties. We use the `?` symbol in a generic type to represent a
type whose type parameter is not specified.

For example, `Option[?]` is a type which represents instances with properties such as `isEmpty` and `toSet`, but
while `isEmpty` will be known to return a `Boolean`, like every other `Option`, the `toSet` property will return
a `Set[?]`; namely, a `Set` containing values of an unknown type. (But, in fact, the _same_ unknown type as the
`Option[?]`!)

The type `Option[T]` also has a `get` method which returns a `T`. But for a wildcard `Option[?]` type, `get`'s
return type is unknown. A wildcard can only appear in type parameter position—not as the entire type—so it's not
possible for `Option[_]#get` to return `?`.

Instead, it returns the most precisely-known concrete type to which the return value will conform. In this case,
nothing at all is known about the type of the return value, so its type is `Any`.

For types with more than one type parameter, like `Either[L, R]`, either, neither or both of the type parameters
may be wildcards. So types such as `Either[?, Int]`, `Either[?, ?]` and `Either[String, ?]` are all possible.

## Conformance

Instances of every different `Option` type, such as `Option[String]`, `Option[List[Int]]` or `Option[Exception]`
all _conform_ to the wildcard type `Option[?]`, which is to say that the values `Some(1)` or `None`
_are instances of_ `Option[?]` just as much as they are instances of `Option[Int]` or `Option[Any]`. So there is
essentially a subtyping relationship between `Option[T]` for any concrete choice of `T` and `Option[?]`.

But the relationship we are calling "subtyping" here is actually subtly more complex. However, these details are
not important right now, and subtyping is an adequate model for almost all purposes. However, it is more correct
to describe this relationship as _conformance_, and we can say, for example, that `Option[String]` _conforms_ to
`Option[?]`.

## Uses for Wildcard Types

None of this would be interesting unless we had good uses for wildcard types. Wildcard types are useful (or even
necessary) for describing sets of instances where,
1. the type parameter is not known,
2. the type parameter is not important, or,
3. the type paramater is not consistent.

## Unknown Type Parameters

There are occasions where the compiler needs to represent an instance of a generic type whose parameter is not
known at compiletime.

This frequently occurs when using reflection on the JVM, because the static types of values cannot be precisely
represented at runtime: an `Option[Int]` can not be trivially distinguished from an `Option[String]`, or indeed
any other type of `Option`. The JVM knows only which _template_ (or equivalently, which _classfile_) was used to
construct that value, and not which type parameters were specified at the time it was constructed.

A wildcard type happens to represent this same known information, while not representing information which is
unknown about the type of the instance in question. A wildcard type represents an erased type with the correct
specificity.

So, in a pattern match, it is correct to write,
```scala
def populated(opt: Any) = opt match
   case x: Some[?] => Success(x.get)
   case _          => Failure(NoValueException())
```
where the scrutinee is tested against the `Some[?]` wildcard type, while it would be _incorrect_ to write,
```scala
def populated(opt: Any) = opt match
   case x: Some[Int] => Success(x.get)
   case _            => Failure(NotAnIntegerException())
```
as it is not possible to verify the type parameter of the scrutinee as `Some[Int]` at runtime.

So while wildcard types represent _imprecision_ in the type parameter of a type, there are nonetheless times
when that imprecision is the exact specificity that is required.

## Unimportant Type Parameters

Often we will write generic methods which take generic types. Here is a method which takes a `List[T]` and
returns its length, if it's non-empty:

```scala
def nonEmptyLength[T](xs: List[T]): Option[Int] =
   if xs.isEmpty then None else Some(xs.length)
```

The signature of this method introduces the type parameter, `T`, which is bound to the type parameter of `xs`, a
`List[T]`. But its return type is invariant, as `Option[Int]`, and its implementation only `isEmpty` and
`length`, neither of which depend on the type parameter, `T`.

We could therefore describe the type parameter of the list as _unimportant_, and hence unnecessary. The same
method could be rewritten with a wildcard type as,
```scala
def nonEmptyLength(xs: List[?]): Option[Int] =
   if xs.isEmpty then None else Some(xs.length)
```
without detriment to its behavior.

## Inconsistent Type Parameters

Sometimes, we need to work with several values all constructed from the same generic type, but with type
parameters in each case.

Consider the generic `Cell` type which can store and retrieve values of a particular type, as well as check
whether the cell is empty or not:

```scala
trait Cell[T]:
   def isEmpty: Boolean
   def update(value: T): Unit
   def apply(): T
```

This may have several instances defined, such as `intCell`, a `Cell[Int]`; `rangeCell`, a `Cell[Range]`; and
`stringCell`, a `Cell[String]`.

If we were to create a `List` of these three `Cell` instances,
```scala
val cells = List(intCell, rangeCell, stringCell)
```
we would like instantiate the `List` with a type parameter which would allow us to use the `isEmpty` method of
each `Cell`—which has a consistent return type of `Boolean`—even though the signatures of the `update` and
`apply` methods differ between different instances.

But because `Cell` is invariant in its type parameter, it is not possible to find a _universal type_ for the
type parameter of `Cell` to which _every_ instance conforms. (Note that this is only true of _invariant_ types,
and we will explore the behavior of _variant_ types later.)

However, in the absence of a universal type, we may choose a wildcard type, `Cell[?]`, to which `Cell[Int]`,
`Cell[Range]` and `Cell[String]` all conform. And furthermore, `Cell[?]` can precisely represent the `isEmpty`
property returning a `Boolean`.

Hence, we can specifically define,
```scala
val cells: List[Cell[?]] = List(intCell, rangeCell, stringCell)
```
and then safely filter the `Cell` instances which are not empty, like so:
```scala
val nonEmptyCells: List[Cell[?]] = cells.filter(!_.isEmpty)
```

Note that it would have been possible to have given the value `cells` the type `List[Any]` or the wildcard type
`List[?]`, but each element would then have been typed as `Any` which would not have represented the `isEmpty`
property, and would have made it impossible to then filter the list's elements using this property.

Wildcard types provide a way of aggregating several types into a single type in a way which precisely represents
the consistent properties, while imprecisely representing the properties that are inconsistent between them.

## Existential types

Wildcard type parameters are a limited form of _existential type_, and they contrast with _universal types_
which cover the majority of types that we work with in Scala.

An existential type parameter, although unknown, is fixed for _each_ instance of that type—it is the type
parameter that was necessarily chosen at the time that instance was constructed, and upon which all its
properties depend—whereas a universal type parameter is fixed for _all_ instances of that type.

This distinction has its foundations in Category Theory, and is the source of much of the elegant symmetry
between different concepts that is ubiquitous in the Scala type system.
