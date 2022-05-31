## Primitives vs Objects

Every value in Scala is an instance of a type. Many values are _primitive values_ or just _primitives_, which
are representations of data that have special, native support within the JVM, and very closely correspond to
operations the computer's CPU is performing. These include number types like `Byte`s, `Double`s, `Float`s,
`Int`s, `Long`s and `Short`s, characters (`Char`s), `Boolean`s and `Array`s of these types.

Other values are _objects_, which can have _state_, itself composed of other values (objects or primitives), and
_methods_ associated with them. Sometimes the values which make up an object's state are called _fields_.
Usually, executing a method on an object will use, in some way, the state associated with the object, maybe
combining it with the method's parameters, and returning a value or changing the object's state.

Methods perform all of the _action_ in Scala code, so the possibilities are literally endless for what they can
do. The idea to bundle methods alongside the state they operate on in objects is one of the fundamental pillars
of object-oriented programming, but it is built on convenience rather than any strict constraints: the values
which form the state of an object may be accessed by the object's methods directly using just that value's
identifier (such as `filename`), without needing to specify the object containing it using "dot-notation"
(for example, `file.filename`). Methods defined in _other_ objects can only access another object's state (or
its methods) by specifying the member's identifier, prefixed with the object's identifier (and the `.`
character).

## Encapsulation

Generally we talk of methods and fields being _inside_ an object, as if the object were a box containing them.
This structure is known as _encapsulation_.

For example,
```scala
object Project extends ProjectDef:
   val tasks: List[Task] = List(Setup, Core, Refinement, Cleanup)
   def todo: Int = tasks.count(!_.done) 

println(s"There are ${Project.todo} tasks to do.")
```

Within the object `Project` we can define a method, `todo`, which refers to the value, `tasks`, directly; we
do not need to reference it as `Project.tasks` (although this is permitted, and has an identical meaning).
Likewise, the definition of `todo` needs to access the `count` method of the list of tasks, but we cannot simply
call `count`; we need the `count` method which operates on the `Task` elements of the `List` called `tasks`,
hence we can reference the `tasks` value directly. Outside of the `Project` object, we _must_ refer to `todo` as
`Project.todo`, or we could, equally, use the expression `Project.tasks.count(!_.done)` directly to get the same
value.

## Object References

Another important benefit to grouping members together in an object may seem very trivial to anyone already
familiar with object-oriented programming, but it makes a significant contribution to the ease of
object-oriented design: a single, atomic reference can be used as a means to access the bundle of methods and
state together, for example:

```scala
def describe(p: ProjectDef): Unit =
   println(s"${p.todo} out of ${p.tasks.length} are incomplete.")

describe(Project)
```

It is clear from the definition of `describe` that, with only a single reference, `p`, passed as a parameter,
the expressions `p.todo` and `p.tasks.length` relate (directly or indirectly) to the state from the same object,
`p`. Accessing a member like `todo` or `tasks` through a reference, as `p.todo` or `p.tasks` is called
_selection_ or _dereferencing_. We _dereference_ `p` and _select_ its `todo` and `tasks` members.

Compare that to a hypothetical alternative implementation which passes the `tasks` and `todo` members as
separate parameters:

```scala
def describe(tasks: List[Task], todo: Int): Unit =
   println(s"${todo} out of ${tasks.length} are incomplete.")
```

It is entirely possible to call this `describe` method with a `tasks` value from one project and a `todo` value
from elsewhere, which would make it possible for `describe` to produce inconsistent output, like this:

```scala
describe(Project.tasks, 10)
```

## Defining Objects

In Scala, we can create a new object using the `object` keyword, with a name for the object. While we can create
a new object almost anywhere, we most commonly want to create an object at the top-level: not nested within
another object, class, method or other structure. Objects created this way are often called _singleton objects_
or just _singletons_ because they are globally unique: there will only ever be at most one instance of the
object inside a running JVM.

That is to say, if we create an object, `BasicLog`,
```scala
object BasicLog:
   val writer: FileWriter = FileWriter("/var/log/application.log")
   def record(message: String): Unit = writer.write(s"$message\n")
```
the identifier `BasicLog` will always refer to the same instance, and likewise, `BasicLog.writer` will always be
the same instance of a `FileWriter`.

## Lazy Instantiation

Unlike `val`s, though, objects are instantiated _lazily_. That means that an object may be defined in our
program, but it will not exist as a "living" value on the heap until one of the running threads makes its first
reference to it. That may happen when the JVM first starts up, if the application's `main` method refers to
`BasicLog` early in its execution, or it may happen after many seconds, minutes or hours, in an application
which can run for a long time without executing certain code.

In the example of our `BasicLog` object, the first time we make a call such as,
```scala
BasicLog.record("Application starting...")
```
will be the moment that the new instance of `BasicLog` actually starts to exist. Because the `Log` object
includes a `FileWriter` as part of its state, that will also be instantiated. In fact, when we write,
```scala
BasicLog.record(msg)
```
it takes nothing more than writing the identifier `BasicLog`, which will return a reference to the `Log` object,
to have the runtime check whether `BasicLog` has been instantiated yet (and if so, return the reference to it)
or start to execute the body of the `BasicLog` object, which also includes instantiating a new `FileWriter` as
part of the `Basic` object's state,

The body of an object specifies what code is executed during the object's instantiation, and, like a method, it
is executed starting from the top, and working downwards. Any `val`s or `var`s in the body will be created as
part of the object's state, and their bodies evaluated, in the order they appear; any method calls will be made,
in the order they appear; any `def`s will be defined as methods on the object, but of course, for a `def`, no
evaluation happens until the method is invoked, so their position in the object's body is not significant.

An object may also contain definitions of other objects, classes, traits and enums in its body, however these
are just definitions, accessed through the object, and do not require any execution while the object is being
instantiated, so, like `def`s, the order they appear in is insignificant.
