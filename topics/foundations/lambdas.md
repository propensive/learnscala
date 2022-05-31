## Anonymous Functions

Functions present methods as objects, making them usable like any other value as parameters and return values,
and generally, within any expression.

These function objects may be constructed automatically from an existing method, or they may be references to a
function stored in a local value or class field. But functions can be written in another form—_lambdas_—which
does not require them to be named, either as values or methods.

Lambdas are _anonymous functions_ which are defined _inline_ at their point of usage. Despite being almost as
capable as functions in defining behavior or functionality, a lambda is not associated with an identifier like a
function is. And since there is no way to refer to a lambda, each lambda will only ever be used at the call site
where it's defined.

The syntax for defining a lambda in Scala can take a couple of different forms, so we will explore the most
explicit variant first.

For example, we can increment every element in a list of integers by `1` using an `increment` function which we
pass to the list's `map` method, like so:
```scala
def increment(x: Int): Int = x + 1

def incrementAll(xs: List[Int]): List[Int] =
  xs.map(increment)
```

In this example, the method `increment` is automatically converted from a method to a function. The `map` method
is able to apply the function once to each element in the list.

But the body of `incrementAll` could be rewritten without the definition of `increment` at all, like so:
```scala
def incrementAll(xs: List[Int]): List[Int] =
  xs.map { x => x + 1 }
```

Here, the method `increment` has been written as a lambda, `x => x + 1`, defined and used at the application
point, rather than as an independent method or function. We typically write lambdas of this form inside
syntactic braces (`{` and `}`) instead of parentheses, (`(` and `)`). And in English, we would read this lambda
example as, "_x_ goes to _x_ plus one".

The behavior of this more concise alternative is the same as the previous version where we first defined
`increment`, but we do not need to define the function in a separate step, or give it a name.

Of course, the value `x` is unknown at the point the lambda is defined, while the purpose of the lambda is that
the function it defines may be applied to values: once or many times to the same value, to many different
values, or perhaps never: the function is just a value, after all.

So the code in the lambda is not evaluated at the point it is defined—and without an instance of `x` it would be
impossible to, in any case—but the function the lambda defines will be instantiated then: an instance of Scala's
function type with an `apply` method that can be invoked with different values. And the method being called with
the lambda may evaluate it, any number of times, and with any parameter it chooses.

## General Lambda Syntax

The general syntax for lambdas takes the form of introducing a named identifier, for example `x`, followed by an
arrow, `=>`, then an expression, written in terms of `x`, which will evaluate to produce the result of the
function being defined.

The identifier `x` is called the _bound variable_ and a different identifier name could have been chosen without
changing the behaviour described by the lambda, so long as the expression on the right side of the arrow adopts
the new identifier name correspondingly. The lambda, `x => x + 1` is identical to `y => y + 1`.

Note that the term "bound variable" is a variable in the sense that it varies for each invocation of the lambda,
while it remains invariant for any particular invocation.

Most lambdas take just one argument, but since methods, and hence functions, can take multiple arguments,
lambdas which take multiple arguments may also be specified, by specifying all arguments as bound variables,
separated by commas (`,`) and surrounded by parentheses, before the `=>` arrow.

For example, a lambda equivalent to the method,
```scala
def div(x: Int, y: Int): Int = x/y
```
would be written as `(x, y) => x/y`.

However, there are limitations to this syntax which are apparent from the lack of types in the lambda syntax, as
compared to its equivalent method implementation which explicitly specifies that both parameters, as well as the
return type, are `Int`s: such a lambda can only be used where Scala is able to infer the types of the bound
variables, `x` and `y`.

Unfortunately, the use of the `/` operator is not sufficient to indicate that `x` and `y` are both `Int`s; their
type could be `Double` or even two different and unknown types, as long as the first type defines a `/` method
that accepts the second type as a parameter!

But thankfully, most of the time a lambda is used in the context of a method call, there is enough type
information available to correctly deduce the types of the bound variables; and from these the return type may
be inferred automatically.

For example, if we were to write, `List(1, 2, 3).map { x => x + 1 }`, the type of `List(1, 2, 3)` is known to be
`List[Int]` and the definition of its `map` method requires a function from the element type to another type,
which is sufficient to infer that `x` in `x => x + 1` has the type `Int`, and hence that `x + 1` also returns an
`Int`.

Yet, if we attempt to define a lambda without any context, like so,
```scala
val fn = x => x + 1
```
then the compiler will complain that it is unable to infer the type of the parameter, `x`. Since the function
object corresponding to `x => x + 1` needs to be instantiated when this code is evaluated (and not at some later
time), the bytecode which implements it must be determined at this point.

The problem may be easily fixed by specifying a type for `x`, and surrounding it in parentheses, like so:
```scala
val fn = (x: Int) => x + 1
```

## Variable Capture

The definition of a lambda represents a function as an object, and the lifetime and usages of that object cannot
be predicted at the time of its creation. After its creation, it may be used once immediately, and never again.
Or it may be stored in a cache, and next accessed several hours after its creation; yet its behavior must be the
same in each case.

The function object therefore needs to hold references to every other object it needs to reference in order to
execute.

For example, consider this code which uses a `Counter` type, whose only purpose is to count the number of times
its `increment` method is called:
```scala
val c: Counter = Counter()

List(1, 3, 5).map { x =>
   c.increment()
   x + 1
}
```

A counter, `c`, is referenced and its `increment()` method is called from within the lambda. So the counter
will be updated every time the lambda is called. For our list of three elements, we should expect that to happen
exactly three times, because we know that is how `map` is implemented, but more generally we don't know the
lifecycle of a lambda.

So at its instantiation, it must capture references to all the objects it may need to access if or when it is
evaluated.

This is achieved silently and transparently by the compiler, and we rarely need to think about it. But for every
object referenced within the body of a lambda, its function object will include a field referring to it, and
which must be passed to the object in order to instantiate it.

For this reason, a lambda which captures references to other objects from the surrounding scope is often called
a _closure_, and is said to _close over_ object references in its enclosing scope. But since Scala performs this
transformation so seamlessly, the terminology does not contribute much to understanding.

## Concise Syntax

The general syntax for writing lambdas may be written in a more concise form for many simple lambdas. If the
bound variable, say `e`, appears exactly once in the body of the lambda it can often be replaced by the
underscore character, `_`, and the binding—for example, `e =>`—may be entirely omited.

For example, the lambda `n => n + 1` can be rewritten as just `_ + 1`, or `str => println(str)` can be rewritten
as `println(_)`.

These forms are almost always preferred, since their intent is clear: `_ + 1` means "add `1` to the value" and
`println(_)` means "print the value", and this style is known as _placeholder syntax_ because the underscore
serves as a placeholder for the bound variable, if we were to imagine it being substituted into an expression.

There is a subtlety to the rewriting rules, which relates to the nesting of the placeholder within parentheses:
A bound variable in the body of a lambda may be replaced by a placeholder _only_ if the usage of the bound
variable is not inside parentheses or braces, _except_ when the bound variable is inside a single pair of
parentheses and constitutes the entire parameter.

So, the first part of the rule permits the following rewritings,
- `x => x + 1` to `_ + 1`
- `c => c.increment()` to `_.increment()`
while,
- `x => (3 - x)/2` cannot be rewritten to `(3 - _)/2`, and
- `str => println(str + "!")` cannot be rewritten to `println(_ + "!")`

However, the second part of the rule permits the rewritings,
- `e => xs.append(e)` to `xs.append(_)`
- `k => calculate(total, k, ratio)` to `calculate(total, _, ratio)`
while,
- `e => xs.append(normalize(e))` may not be rewritten to `xs.append(normalize(_))` because the bound variable is
nested inside two pairs of parentheses.

Furthermore, lambdas with two or more bound variables may also be rewritten using placeholder syntax, provided
the rules above are followed for each bound variable, and the each bound variable appears exactly once in the
lambda's body, syntactically in the same order when read from left to right.

So, we can rewrite `(xs, e) => xs.append(e)` as `_.append(_)` and `(k, q) => k/q` as just `_/_`.

But placeholder syntax cannot be used if the parameter order needs to change. The lambda, `(q, k) => k/q` has
no equivalent using placeholder syntax.

The combination of lambdas for defining functions inline and placeholder syntax can very efficiently express our
intent when working with Scala code which takes functions as parameters.

