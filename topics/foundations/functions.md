## Functions as Values

One of the most important features of Scala, as a functional programming language, is its ability to treat
functions as first-class values.

This can seem unintuitive to begin with: surely a function is something which operates on values, not a value
itself? But actually functions in Scala have all the properties necessary to be values: they can be passed as
parameters to other functions; they can be returned from other functions; they can be stored in fields in a
class or object.

## Types of Function Values

In Scala, we can write the type of a function value using the `=>` arrow between its parameter type (or input
type) and its return type.

For exaple, a function which converts an integer to a string (such as `Int#toString`) may be written as,
`Int => String`. Or a function which reads a file and returns a list of search matches might have a type such
as, `File => List[Match]`.

The type tells us only the input and output types of the function value. It provides no information on what the
function actually does. In particular, the type says nothing about which other values it accesses, whether it
produces any side-effects, or whether it is deterministic.

So a function's type is a black box around its behavior.

This has both advantages and disadvantages. Often we would like the certainty from a function's type that it
does not produce any side effects, but that is not available in Scala. Yet at the same time, by specifying a
function's type only in terms of its input and output types, we allow _any_ function (representing _any_
behavior) to be used in a position with that same type, so long as it accepts the same types of input and
returns the same type of output.

## Some Examples

Let's consider an example. If we first define a couple of methods,
```scala
def removeSpaces(str: String): String = str.replaceAll(" ", "")
def allCaps(str: String): String = str.toUpperCase
```
we can then assign them to values, like so:
```scala
val removeSpacesFn = removeSpaces
val allCapsFn = allCaps
```

These values, `removeSpacesFn` and `allCapsFn`, are both values and functions. Their right-hand sides are
references to the methods which define their implementations, but importantly they are _unapplied_, meaning that
they are lacking any parameters (`String`s in these examples) that would be necessary for them to be evaluated;
and so they also remain _unevaluated_.

Written with their types, these definitions would read,
```scala
val removeSpacesFn: String => String = removeSpaces
val allCapsFn: String => String = allCaps
```
and would behave exactly the same in expressions as the methods that were used to define them: the expressions
`allCaps("hello")` and `allCapsFn("hello")` will evaluate to the same values.

But, as values, we can use them in ways that we could not traditionally use a function. Here is a contrived
example which defines a new function in terms of `removeSpacesFn` and `allCapsFn`,
```scala
def changeString = if math.random < 0.5 then removeSpacesFn else allCapsFn
```
which we can then apply to a string, as `changeString("Hello World")`, which would randomly return either
`"HelloWorld"` or `"HELLO WORLD"`.

Assigning `removeSpaces` and `allCaps` to values first was, in fact, not a prerequisite for such a definition,
so it's possible to skip this step entirely, and just write:
```scala
def changeString = if math.random < 0.5 then removeSpaces else allCaps
```

Without this ability to treat functions as values, we would have to define an equivalent `changeString` as,
```scala
def changeString(str: String) = if math.random < 0.5 then removeSpaces(str) else allCaps(str)
```
where every method can only be used when its parameters have been applied.

## Methods vs Functions

There is a subtle difference in Scala between two separate concepts—functions and methods—which is often
unimportant since it is mostly an implementation detail, but which we should nevertheless be precise about.

Both methods and functions define some computational behavior on their inputs to produce outputs. They can be
composed in expressions and evaluated to produce results.

While functions are always values, methods are not: they are named members of a class or trait (and hence, of
instances of classes or traits) which, at a low-level, fundamentally define the instructions (in bytecode on the
JVM) that operate on other values.

But methods may be seamlessly converted into functions, which Scala will do—silently—whenever necessary, so the
distinction between them becomes less apparent.

Every function is a value which is an instance of a subclass of a built-in function type whose abstract `apply`
method defines the function's behavior. To convert a method to a function, Scala will create an anonymous
subclass of its function class, implementing its `apply` method with code that calls the original method being
converted. It then instantiates the new subclass with a reference to the object that the original method is a
member of—without which it would not be possible to refer to the method!

What this produces is a function which has the same behavior as the original method, but it is now a value; an
object which is an instance of Scala's function type, meaning that it conforms to a very stardard interface:
regardless of the name of the method used to create the function, its functionality is now being presented
through a method called `apply` on an instance of a standard type.

Scala uses a variety of different classes to represent different types of function. Most commonly it uses
`Function1`, and the type `A => B` is a more legible alias for `Function1[A, B]`.

The `1` in `Function1` specifies the number of parameters the function takes as its input (its _arity_), and
several different function types exist for different sorts of functions exist for representing methods with
different signatures. All methods (and hence functions) return a single value.

This means that methods, like the `map` method of `Option` or `List`, can be written which accept _arbitrary_
functions as parameters, so long as those functions represent methods with the correct signature. While the
detail usually remains hidden from us, such methods are implemented by calling the `apply` method defined on the
function instances passed to them, without any knowledge of the names of the methods used to construct them.

Scala will enforce additional constraints on the input and output types of functions, to ensure that it's never
possible to call a method (via a function) with a parameter that it cannot accept, nor to have a function return
a value of a type that is not expected.

## Summary

The capability to treat functions as just another kind of value means that we do not need to learn different
rules for working with functions versus values, and all of the machinery—the typesafety, the linguistic
expressivity and flexibility—can apply equally to function values.

Indeed, functions have additional features that other values do not—they can be applied to other values, of
course—but by treating them as values, we have just one concept to understand.
