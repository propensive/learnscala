## Doing Things at the Same Time

The JVM is capable of doing more than one thing at the same time. We can write software which can respond to
multiple HTTP requests at the same time, starting to serve one request before the previous one has completed, or
we can start processing the start of a file while we are still reading it from disk, at the same time as writing
its transformed content to a new file on disk.

This is possible for the same reason it is possible for an operating system to multitask: modern computer
systems usually have multiple CPUs cores, with two, four or eight cores common, and each CPU core able to work
on a different task at the same time.

Furthermore, when there are more active concurrent processes than processors to run them on, operating systems
are able to give the appearance that all processes are running concurrently by pausing and switching between
them frequently enough that we can't easily observe that they are not all running continuously, all the time.

This is the subtle distinction between _concurrency_, when the start and end times of many tasks can overlap,
and _parallelism_, where multiple tasks are being executed at the same instant in time.

## The Threading Model

There are many different paradigms for modeling the execution of more than one operation concurrently, and many
frameworks and libraries exist that aim to make it easy to write code with these models, with different
attention to programming ease or performance. But, for any code running on the JVM, support for concurrency is
provided through the low-level construct of a _thread_, even when higher-level abstractions are provided to hide
this detail from us.

There may be just one or there may be thousands of threads in a running JVM, though the overhead of switching
between them puts an effective limit on how many it is possible to have running concurrently. Later versions
of the JVM promise to dramatically reduce this overhead, so if we look towards the future, we should not be too
concerned about this, but it is almost always desirable to keep the number of active threads in a running JVM
low.

A JVM always starts with a single main thread, which may _fork_ or _spawn_ new threads as desired. Each thread
has a name and an execution stack, and once started, it will run until it finishes its work, at which point it
may be _joined_ to the thread that started it. Threads will therefore come in and out of existence.

But with the exception of the main thread, each running thread exists because it was forked from another thread.

## An Example

We can create a new thread by defining a class extending `Thread` which implements the `run()` method to specify
the thread's behavior.

```scala
class NotifyThread(name: String) extends Thread:
   override def run(): Unit =
      for i <- 1 to 10 do
         println(msg)
         Thread.sleep(1000L)
```
and starting instances of it, like so,
```scala
val thread1 = NotifyThread("Hello...")
val thread2 = NotifyThread("...world!")

thread1.start()
Thread.sleep(500L)
thread2.start()

thread1.join()
thread2.join()
```

The `Thread.sleep` method pauses the current running thread for a number of milliseconds.

So, this code will create two new threads, each of which will print a (different) message ten times, with 1000ms
between each message. The second thread is started half a second after the first, to try to make their messages
roughly evenly spaced between the messages from the first thread. Finally, both threads are joined to the main
thread, which waits for them to finish, then ends their existence.

## Independence, not Isolation

Each thread will operate independently of the others, thanks to having its own dedicated stack which keeps track
of its execution. This is _independence_, but not _isolation_, as threads may share references to the same
objects and may access and modify the same heap memory locations.

This capability is both powerful and dangerous. Very frequently we want to check a condition that is dependent
on a value, then perform another operation using that value.

```scala
object Data:
   var list: List[Int] = initialList

def pop() =
   if !Data.list.isEmpty then
      println(Data.list.head)
      Data.list = Data.list.tail
```

Here, the `pop` method checks that the global mutable variable, `Data.list` is not empty, then gets its head,
prints it, and finally updates the list to remove the head. This appears to be reasonable, and if we could
guarantee that only one instance of `pop` could ever possibly be running at any time, it would also be safe.

But what if two different threads were both calling the `pop` method concurrently? Both methods access have
access to the same `List[Int]` on the heap, and both can modify it. The value of `Data.list` could be,
`List(42)` when one thread starts executing `pop`, but in the short time between checking the list is not empty,
and attempting to get the head (in order to print it), another thread could mutate the variable to change it to
the empty list, `Nil`, and our call to `Data.list.head` would fail (with an exception).

The same could happen one step later within the method, and between printing the list's head, another thread
could mutate the list to replace it with its own tail, meaning that one head element would fail to be printed.

It may seem unlikely that for a method which runs so fast, that two threads could be timed to interact so badly,
and that is true, but if we increase the number of concurrent threads calling `pop()`, run them in a tight loop,
or run them for a very long time, and it becomes almost certain that it would occur at least once, and just once
in a thousand times would still be a bug.

## Synchronization

It is easy to look at the definition of `pop` and assume that it is safe. But it would be a mistake to apply
only _local reasoning_ to make that judgement, when we are working with global mutable state. With practice, we
can learn to avoid mistakes like this, by not writing methods which use global mutable state, and by more
carefully identifying when it can be misunderstood. But this example illustrates how easy it is, and how aware
we need to be about the power threads give us.

One simple fix offered by the JVM is to _synchronize_ the method. This provides a guarantee that the
synchronized code will never be run concurrently, that is, only at most one instance will be active at any time.
Synchronization uses a _lock_ associated with an object which should, at all times, indicate whether any other
thread is using the lock, and if so, will wait until it becomes available. We use the `synchronized` method,
which is available on every JVM object, to apply this guarantee.

```scala
def pop() = Data.synchronized {
   if !Data.list.isEmpty then
      println(Data.list.head)
      Data.list = Data.list.tail
}
```

We use the `Data` object's lock, here, though any global object would, in fact, be acceptable.

## Deadlock

Synchronization can help to avoid certain concurrency issues, but it is is not without its own risks.
Synchronized methods are susceptible to _deadlock_ where a synchronized method holds a lock, and the release of
that lock is dependent on code which needs to take the lock.

The lock cannot be taken until it is released, and it cannot be released until the code elsewhere has taken it.
This risk can be avoided with care, but casual overuse of synchronization can make deadlock a little too easy
for comfort.

Threads give the JVM enormous power to perform many operations concurrently, which is almost essential for
a modern programming environment. They are, however, a low-level construct which can make our code less safe,
and they often serve us better as the foundation for higher-level concurrency models. Nevertheless, whatever
concurrent programming methods we employ, they will be built upon threads.
