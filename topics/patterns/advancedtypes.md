Working with nested classes can lead to path-dependent types. For example, imagine a representation of a
form containing fields.
```scala
class Form(name: String):
   private var fields: List[Field] = List()
   class Field(id: String, label: String, conceal: Boolean = false)
```

We may choose to construct some new `Form`s with some `Field` instances, like so,
```scala
val signup = Form("Signup")
val name: signup.Field = signup.Field("name", "Full name")
val email: signup.Field = signup.Field("email", "Email address")

val login = Form("Login")
val username: login.Field = login.Field("username", "Username")
val password: login.Field = login.Field("password", "Password", conceal = true)
```

The JVM treats inner classes such as `Field` like any other class, so every `Form##Field` will be compiled to an
instance of the same class, referred to internally as, `LForm$Field`. That means that both `signup.Field` and
`login.Field` will be instances of the same _runtime_ class, even though the compiler can distinguish them at
compiletime.

In this example, we have four fields, belonging to two different forms. If we were to create a `List` of all
fields,
```scala
val fields = List(name, email, username, password)
```
its type would be `List[Form##Field]`, that is, a list of form fields, where we do not know which particular
`Form` instance—`signup` or `login`—they relate to. This is reflected in the projection type, `Form##Field`,
which is a supertype of both `signup.Field` and `login.Field`.

But what if we wanted to collect the IDs of the form fields relating to the `signup` form from our list? We
might be tempted to write,
```scala
fields.collect { case field: signup.Field => field.id }
```
but if a `signup.Field` and a `login.Field` are instances of the same _runtime_ class, would this not collect
the IDs of the `login` fields too?

Thankfully not, but it requires some explanation why!

While the runtime type of each field can disambiguate a `Form##Field` from other simple types using an
`isInstanceOf` check, this does not offer a way of distinguishing a `signup.Field` from a `login.Field`. But,
being an inner class, it contains a hidden reference (whose name is `$outer`) to the instance of the outer
class used to construct it. This reference provides everything we need. The compiler will include an equality
check between the `Field`'s `$outer` reference and the path in the path-dependent type.

So a match such as,
```scala
(field: Form##Field) match
   case _: signup.Field => 1
   case _: login.Field  => 2
   case _               => 3
```
would be compiled to,
```scala
if field.$outer eq signup then 1
else if field.##outer eq login then 2
else 3
```

Note that the `field` instance is already statically known to be an instance of `Form##Field`, so there is no
need to perform the `field.isInstanceOf[Form##Field]` check.

So Scala allows us to pattern-match on path-dependent types, but only by taking advantage of additional
information about path-dependent types.
