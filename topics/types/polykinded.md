## Matching Kinds

Usually when we use a type of some kind, whether that's a one-parameter type constructor like `List`, a
two-parameter type constructor like `Either`, or a proper type, such as `Ordering[Int]`, the kind of the type at
the use-site must match the kind of the type expected at the definition site.

More concretely, we can, for example, call a method,
```scala
def action[F[_]](): Unit = ()
```
with `action[List]()` or `action[Ordering]()`, but not `action[Map]` or `action[String]`, since `Map` takes two
type parameters, and `String` takes none.

This constraint is quite reasonable, since the implementation of `action` will very likely make use of proper
types such as, `F[String]` or `F[List[Int]]`, but both `Map[String]` and `String[List[Int]]` would be
nonsensical types.

An error will be produced at compiletime whenever a type of one kind is used in a type position where a type of
a different kind is expected. By analogy to typesafety, we can say that Scala is a _kindsafe_ language.

## Philosophy of Kinds

The usage of proper types is quite easily understood in Scala: these are the types which describe the properties
of values, and which can be ascribed to values in fields, parameters and return types, or emitted in bytecode
(even though that may be in an erased form).

Type constructors and higher-kinded types, however, cannot be used in this way, and given that the end-product
of compilation is bytecode (which can contain _only_ proper types) they are not useful, at least in a direct
sense, on their own. An improper type must—at some point before bytecode is emitted—be assimilated with
another type to produce a proper type. Every other usage of improper types is relevant only during compilation.

So we can view improper types as somewhat ephemeral entities that we introduce into our code only so that they
can be eliminated later, since they cannot be directly used in any useful way at runtime.

### An Analogy

In this sense, they can be thought of as playing a similar role to negative numbers in simple arithmetic. In the
material world, everything that we see around us, such as lengths, masses and numbers of things, can be
expressed using only positive numbers. And (if only we were not already very familiar with negative numbers!) it
might be hard see how negative numbers arise _naturally_.

But we can still express a problem only in terms of positive numbers to which the solution requires negative
numbers. For example: "What is the number to which we can add 10 and get the answer 3?"

It can be hard to imagine what the answer, -7, really means in a _tangible_ sense, in part because we're limited
by our experience of the real world, in which only positive numbers are exhibited directly. But -7 is
nonetheless the number we _invent_ to provide a solution to the problem, and our only explanation without it
would be that our problem simply has no solution. And that might be unsatisfactory!

But the introduction of a negative number to solve a problem is only a _temporary_ departure from the
familiarity of positive numbers. Since our problem space is defined—and our solution space must be defined—only
in terms of positive numbers, negative numbers serve us only to "bridge" an otherwise-awkward gap.

And the same can be said of improper types: we represent them in the language only so they may be combined with
other types (proper or improper) to produce proper types.

Consider the higher-kinded type, `Functor`, which has the kind, `_[_[_]]`. It may be combined with a type
constructor such as `Set`, which has the kind `_[_]`, to produce the proper type `Functor[Set]`—which is a type
we can instantiate!

And that instantiation process will involve providing all the types necessary to remove the holes in a template
so that every method and field definition is fully-specified.

## Kind Polymorphism

So it may seem unlikely that we would find utility in _kind polymorphism_, sometimes known as
_polykinded types_: abstract types whose kind is not known at the definition type, and which may be proper or
improper.

Types of a known kind can be combined with other types of a known kind to produce proper types, but in the case
of a polykinded type, we cannot be certain (for as long as that type is abstract) what kinds of types it could
be combined with (if any at all).

### An example

We can express kind polymorphism in the definition of any type parameter as follows:
```scala
def task[Z <: AnyKind](): Unit
```

Indeed, if we give the abstract type, `Z`, the bound `<: AnyKind`, we immediately lose the ability to use it in
almost any way we would normally be able use a type!

It's not possible to specify any parameter of `task` in terms of `Z`, nor the method's return type. And the same
applies to the method body of `task`. That's true, except when `Z` appears in a type parameter position which is
also polykinded.

We can therefore define a typeclass with a polykinded type parameter, such as,
```scala
trait TypeName[T <: AnyKind](val name: String)
```
and use it as a context bound on `Z` in the definition of `task`, like so:
```scala
def task[Z <: AnyKind: TypeName]: String =
  s"Task for ${summon[TypeName[Z]].name}"
```

Now, with a few `TypeName` definitions in scope, such as,
```scala
given TypeName[List]("List")
given TypeName[String]("String")
given TypeName[Either]("Either")
given TypeName[Functor]("Functor")
```
it becomes possible to call,
```scala
task[List]
```
to return the string `"Task for List"` or,
```scala
task[Functor]
```
and produce `"Task for Functor"`, with a consistent method call whose type parameter is not constrained by any
particular kind.

We could also, of course, try to call `task[Functor[List]]`, but unless a given `TypeName[Functor[List]]` exists
(or a proper-kinded supertype) in scope, it would not compile due to the missing contextual value. `Functor` and
`Functor[List]` are not only different types, but different types with different kinds.

### Mechanism

This works, primarily, because the typeclass `TypeName` serves to eliminate the polykinded type, by providing an
instance that conforms to a known interface—the `TypeName` trait—and which can facilitate provision of any
number of values, methods, type members with known kinds, and other members, for use in the method.

The `TypeName` example did not appear to be particularly exciting, since we used it only to derive a
straightforward `String`. But the nature of a polykinded type is that it is so general that we can do little
more than infer a value from it.

We might, for example, like to define an extension method on lists of pairs which takes a type constructor as a
type parameter, and returns an incarnation of that type, while allowing the type constructor to have any number
of parameters.

So if `xs` is a `List[(Int, String)]`, we would like to be able to call `xs.as[Set]` and `xs.as[Map]`, even
though `Set` takes one type parameter, and `Map` takes two.

First, we should consider the return types. If converting to `Set`, then `Set[(Int, String)]` would be a
reasonable return type, while converting to `Map` should produce a `Map[Int, String]`. Clearly these types must
be constructed in different ways, depending on their kinds.

Our extension method definition (without an implementation, yet) would look similar to this,
```scala
extension [K, V](xs: List[(K, V)]) def as[T <: AnyKind] = ???
```
but in order to provide different functionality for `Map` and `Set`, we would also need to introduce a
typeclass, which we will call `Converter`, also parameterized with a polykinded type parameter.

Here is an incomplete definition for that typeclass interface,
```scala
trait Converter[T <: AnyKind]:
  def convert[K, V](xs: List[(K, V)])
```
where the return type of the `convert` method has not been specified.

This enables us to implement the extension method as,
```scala
extension [K, V](xs: List[(K, V)])
  def as[T <: AnyKind](using conv: Converter[T]) = conv.convert(xs)
```
but we still need to specify the return type.

Since our method is generic, that return type must also be dependent on the type parameters, `K` and `V`, but
the way those type parameters are composed into the return type will be dependent on the type `T`. Yet, at the
same time, we need to present that through a consistent object-oriented interface.

And it can be achieved by adding an appropriate type member to the typeclass. Let's update the definition to,
```scala
trait Converter[T <: AnyKind]:
  type Return[K, V]
  def convert[K, V](xs: List[(K, V)]): Return[K, V]
```
and provide `Converter` instances for `Set`,
```scala
given Converter[Set]:
  type Return[K, V] = Set[(K, V)]
  def convert[K, V](xs: List[(K, V)]): Set[(K, V)] = xs.to(Set)
```
and `Map`,
```scala
given Converter[Map]:
  type Return[K, V] = Map[K, V]
  def convert[K, V](xs: List[(K, V)]: Map[K, V] = xs.toMap
```

We can also be explicit in our extension method's return type, specifying it as dependent on the `conv`
parameter:
```scala
extension[K, V] (xs: List[(K, V)])
  def as[T <: AnyKind](using conv: Converter[T]): conv.Return[K, V] =
    conv.convert(xs)
```

Together, these definitions provide the desired syntax, and calling `xs.as[Map]` will return a `Map`, and
calling `xs.as[Set]` will return a `Set`.

But perhaps curiously, there is no requirement for the specified type parameter to be related in any way to the
return type of the `as` method!

That return type is specified as the right-hand side of the `Return[K, V]` type member within each typeclass
instance, but the choice is not dependent on the polykinded type parameter except by virtue of being defined in
its corresponding given instance: it is only by convention that we have specified return types which feature the
polykinded type parameter. In each case, we could have returned an entirely different type.

And this is because, again, there is simply no way to assimiliate a polykinded type with another type. We can
only introduce a relationship between a polykinded type and a type whose kind is known through a typeclass
interface.

## `AnyKind`

The type `AnyKind` exists in the type hierarchy as a supertype of `Any`, even more general than `Any` as it is
not even specific about its kind (whereas at least `Any` is known to be a proper type).

This is a slight special case in Scala's syntax, since, by default, any type parameter's upper bound will be
`Any`. But `AnyKind` is a _supertype_ of `Any`, and under normal circumstances, in the presence of two distinct
upper bounds on a type, the more precise of the two would be chosen. Yet, when `AnyKind` is specified, instead
of being _subsumed_ into `Any`, it _overrides_ the default upper bound of `Any`, widening the type parameter
relative to its default.

The type `AnyKind` itself has no members, since these cannot even exist on improper types.

## Summary

Kind polymorphism is an advanced and niche feature, but it enables a certain flexibility of syntax which would
otherwise be difficult to provide.
