FIXME: This text was cropped from enums2.md, and should be incorporated into a later lesson.

At the same time, though, we need the value `Empty` to be a valid `Maybe[Char]` and a valid `Maybe[Int]` and a
valid `Maybe[T]` for any type `T`, which is to say, we need its type to be a subtype of _all_ of these types.

By default, a type parameter of any template is _invariant_, which means that every instance of that type must
have been constructed from a template using _exactly_ the type specified, and not a _related type_ such as a
subtype or supertype.

Consequently, if `Maybe[Char]` represented all the values which were constructed with `T` equal to `Char` in
the `Maybe` template, and `Maybe[Int]` reperesented all the values which were constructed with `T` equal to
`Int` in the same template, it would be impossible to find a `T` which conforms to both `Maybe[Char]` and
`Maybe[Int]` which we could use for the type of the value `Empty` which is, after all, a singleton value.

While it would, in theory, be possible to construct many different instances of `Empty` for each and every type
of `Maybe`, this would be unnecessary and would waste memory.

Instead, Scala's enumerations use just a single value for every singleton case in an enumeration, but require
that any type parameters which do not appear in _every_ case in that enumeration are made _variant_. The
definition of `Maybe` above does not meet that requirement, so it will not compile.


Typically, the solution is to make the type parameters covariant by specifying a `+` immediately before them in
the definition, as in this definition of `Option` as an enumeration:

```scala
enum Option[+T]:
   case Some(value: T)
   case None
```

Unlike a type with an invariant type parameter, a type with a _covariant_ type parameter represents instances
which were constructed with a choice of `T` equal to, or a subtype of, the specified type.

For example, the type `Option[Throwable]` can represent instances constructed as `Option[Exception]` and
`Option[Error]`, because `Exception` and `Error` are both subtypes of `Throwable`, as well is instances
constructed as `Option[Throwable]`.

And this additionally provides a solution for the type of `None`: it becomes an `Option[Nothing]`. Because the
type parameter of `Option` is covariant, the relationship between `Option[T]` and `Option[S]` will mirror the
relationship between `T` and `S`. If `T` is a subtype of `S`, then `Option[T]` is a subtype of `Option[S]`. If
`T` has no subtype or supertype relationship with `S`, then `Option[T]` will also have no subtype or supertype
relationship with `Option[S]`.

But `Nothing` is a subtype of _every_ type, `T`, therefore `Option[Nothing]` is a subtype of `Option[T]` for
_every_ type, `Option[T]`. And therefore, the value `None`, with the type `Option[Nothing]` conforms to every
`Option[T]`.

