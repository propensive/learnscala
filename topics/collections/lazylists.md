## Lazy lists

Scala's `List` type provides a way to define sequences of elements as either the empty list, called `Nil`, or as
the "cons" of one "head" element to a "tail" which is just another `List` instance, thereby allowing sequences
of arbitrary length.

While the length of a `List` instance may be _arbitrary_, it is nevertheless fixed and finite: every "cons"
must include a tail which is a reference to another `List` existing on the heap, and is immutable, so after its
creation, it's not possible to change in any way.

The are some alternative mutable collections in Scala's standard library which can be modified after their
creation, but their mutable nature can bring its own challenges.

There does exist, in fact, a data structure which is both immutable and whose elements need not be fully
specified at the moment of its construction: `LazyList`.

A data structure which combines immutability with the lack of a full specification may seem like a
contradiction, but it is not! A `LazyList` is conceptually and structurally very similar to a `List`, but its
value—either an empty `LazyList` or the "cons" of a head value to a `LazyList` tail—is not calculated until it
is needed.

So effectively, a `LazyList` is a reference to a function which, the first time it is accessed, will calculate
the `LazyList`'s head and tail member values for that instance (or could potentially calculate the empty
`LazyList`). On subsequent accesses, the function will return simple references to these values, without any
calculation.

So, the head and tail of a `LazyList` may not be available instantaneously at the moment they are accessed,
since they may need to be calculated (and that may rely on time-consuming computations, or simply waiting for
some more data to become available), but once calculated, they can never change or be recalculated to different
values. And this guarantees immutability.

We should pay close attention to what it means to calculate the tail of a `LazyList`. In general, that does
_not_ mean computing every remaining element in the sequence; it just means constructing the `LazyList` instance
for the tail. And we shouldn't forget that that `LazyList` is, of course, lazy itself.

So calculating the _tail_ of one `LazyList` essentially requires only the next element in the sequence to be
calculated, because the _tail of the tail_, and every tail thereafter, is also lazy and will be computed only
when necessary.

## Uses of `LazyList`s

This subtle change in the way the `List`'s head and tail are specified in a `LazyList` introduces a wealth of
possibilities that are not available to us in a strictly-computed data structure like `List`.

The elements in a `LazyList` may depend on state which is not known (or not knowable!) at the time it is
constructed, or which changes independently of the `LazyList` instance, such as user input from a keyboard or
the network. The ultimate length of a `LazyList` may be affected by events, or may even be effectively infinite.

We can construct an endless `LazyList` using the `LazyList.continually` method, like so:
```scala
val ones = LazyList.continually(1)
```

Operations on `LazyList`s will compute as much of the sequence as necessary. So an operation such as accessing
the seventh element, `ones(7)`, will compute just the first seven elements of the `LazyList`, while calling
`isEmpty` only needs to compute the current `LazyList` to ensure that there is at least one more element.
Calculating the `length` of a `LazyList` requires the entire sequence of elements to be calculated, since
there's no other way to know after how many elements the sequence terminates.

Given an endless `LazyList` such as `LazyList.continually(1)`, we can still make it finite using certain
collection methods. For example, `LazyList.continually(1).take(100)` will create a sequence which ends after one
hundred elements, or for another stream, `xs`, we might be able to terminate it when we receive a certain "stop"
signal, for example,
```scala
val elements = xs.takeWhile(_ != Stop)
```

What is perhaps interesting about these methods is that they are also lazy, so they return results almost
instantaneously, without needing to compute the entire sequence up to the point where it terminates.

## Streaming

This makes `LazyList`s particularly suitable for streaming data, since it allows an immutable data structure to
provide an arbitrary-length stream of _events_: elements that occur at a particular time, and hence, appear in
an order.

So we can process a stream of events using exactly the same methods we would use on a `List`, while keeping in
mind the operation may take a long time to complete: seconds, minutes, hours, days—or even theoretically
forever!

Here's a simple example of processing an event stream,
```scala
val stream: LazyList[String] = eventStream()
stream.foreach { event =>
  println(s"Received event: $event")
}
```
where each received event (just a `String`, here) is printed as a side-effect.

But using a more functional style, a sequence of events could also be used to update some state, using a fold
like so:
```scala
val stream: LazyList[Event] = eventStream()
stream.foldLeft(initialState) {
  (state, event) => state.updated(event)
}
```

In either case, the code will look very similar for either a `List` or a `LazyList`.

However, in the latter case, while the stream is being calculated, the current thread will be consumed,
calculating the next value, and potentially blocking. So while the value `stream` looks as much like a simple
data structure as `List` does, it actually encapsulates a series of computations that will be executed—lazily,
of course—on the current thread.

But at no point do we need to worry about mutation: values may take a while to become available, but once they
do, there is no opportunity for them to change.

## Garbage Collection

The examples above may be subtly problematic in many cases, though!

The nature of `LazyList`s as highly suitable for stream processing, and the potential for those streams to be
long—maybe *very* long—introduces a risk that the stream could become too large to hold in memory at the same
time.

But by this same nature, once each event in the stream has been "processed"—whatever that means in a particular
application—references to those events no longer have any use. The nature of an event is that it is processed
once, and thereafter we care only about the event's consequences, not the event itself.

Yet, our examples included a reference to the stream,
```scala
val stream: LazyList[Event] = eventStream()
```
which is an immutable data structure.

And this means that its `head` element will always be the same first event; its second element will also be the
same; and likewise, every other element will be the same, regardless of whether they were accessed shortly after
the program was started, or days later.

The problem is subtle, though. The reference to the `LazyList`, accessible through the `stream` identifier, is
exactly that: accessible. Our program is able to refer to the identifier `stream` at any time, for as long as
the scope in which it is defined continues to exist, and it may use that reference to reach any event in that
`LazyList`. That means the entire sequence, and every heap object indirectly referenced by it, must be kept in
memory.

That could total many megabytes or even gigabytes of data, and is an example of a _memory leak_.

Depending on the nature of the scope in which the reference appears—which may be a class body, method body,
lambda body, or even a block within an expression—its lifetime may be inherently associated with stack frame
(i.e. a method invocation) or a heap object, and references to `LazyList` instances within any long-lived method
or object are those that are likely to be problematic.

The solution, unfortunately, is equally subtle. We must guarantee that references to the entire stream are never
accessible; we must ensure that no single reference to the original `LazyList` instance can be called from any
part of our program.

The _absence_ of any such reference makes it possible for the JVM's garbage collector to safely remove not just
the `LazyList` references from memory, but also the objects they reference in their `head` fields.

For this reason the usefulness of `LazyList`s for streaming data relies heavily upon the ability of the JVM's
garbage collector to remove the "used" part of a stream from memory once it is no longer required.

We could rewrite our last example without `val stream` to avoid a memory leak, like so:
```scala
eventStream().foldLeft(initialState) {
  (state, event) => state.updated(event)
}
```

We rely on the (hypothetical) implementation of `eventStream()` to _return_ a `LazyList` instance without
_storing_ a reference to that instance anywhere else before returning it (at which time, the scope of
`eventStream()`'s method body ceases to exist). And we can observe that it would be impossible, in our rewritten
example, to obtain a reference to the instance returned by `eventStream()`: there is simply no identifier we can
use to refer to it!

## An Experiment

This difference may be observed quite easily with an example.

We can define an infinite `LazyList` where every element is large enough that too many of them will quickly fill
the entire heap. Here we define an endless sequence of 1MB array instances:
```scala
def endless: LazyList[Array[Byte]] =
  LazyList.continually(new Array[Byte](1000000))
```

Every reference to `endless` will construct a new, different `LazyList` instance, as it is defined using `def`.
But we could have defined it as a `val`, thus:
```scala
val endless: LazyList[Array[Byte]] =
  LazyList.continually(new Array[Byte](1000000))
```

In both cases, calling `endless.foreach(println(_))` will print each `Array` instance's string representation,
for example,
```
[B@5c79c5ec
[B@34eaf9c1
[B@5bbf744b
...
```
but while the first version will keep printing for a long time, the second will quickly fail with the exception,
```
java.lang.OutOfMemoryError: Java heap space
```
since the reference, `val endless`, cannot be garbage collected, and nor can any of its 1MB elements.

Conversely, the usage of `endless` in the expression `endless.foreach(println(_))` will cause the a new
`LazyList` to be constructed, but it may be garbage collected almost as soon as it is created, so long as no
references to it remain accessible anywhere.

But we must remain vigilant for accidental references. If we were to refactor the expression,
`endless.foreach(println(_))`, with a `printAll` helper method, like so,
```scala
def printAll[T](lazyList: LazyList[T]): Unit =
  lazyList.foreach(println(_))

printAll(endless)
```
that might seem like an innocuous change, and there is no explicit `val` referencing the `LazyList`, but the
parameter `lazyList` to the method `printAll` is nevertheless a reference to the entire `LazyList`, defined in
the body of the method `printAll`. Since the scope of `printAll`'s method body lives for as long as the call
to `LazyList#foreach` is executing, the reference to `lazyList` cannot be garbage-collected until it completes.

Unfortunately, for our `endless` stream, the only way it will complete is exceptionally, with an
`OutOfMemoryException`!

## Summary

This reliance on the _absence_ of a stored reference, even if that reference has no bearing on the correctness
of our code (since assigning `endless` to a `val` or a `def` appear to be semantically identical, if there is
just a single usage), may seem unsatisfactory.

And that's not an inappropriate response! It remains one of the challenges of working with streams, and to some
degree, the simplicity gained from choosing an immutable data structure over a mutable data structure is lost
from the ease by which an accidental reference can cause a memory leak, and ultimately produce less reliable
software.

Different application requirements and demands may tip the balance towards choosing a mutable approach on some
occasions and an immutable approach, others; both have tradeoffs.

But in any case, Scala accommodates both.
