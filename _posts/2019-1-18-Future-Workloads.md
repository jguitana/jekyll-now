---
layout: post
title: Scala concurrency 1: insights on Future
---

One of the most relevant Scala primitives for concurrency is the `Future`. It represents a value which is either complete or failed, and is scheduled to run outside the program main flow, in some other thread. You can learn more about it [here](https://docs.scala-lang.org/overviews/core/futures.html) if you are not familiar and want to understand this post.

Future enables us to tackle the following concurrency problems:
1. Let other components execute when one is waiting on IO.
2. Make small programs communicate to build larger ones.
3. Use all the CPU's in our machines.
4. Enable human-computer interaction (GUIs, terminal).

In this post we will look at three distinct kinds of workloads we can run in a Future: `cpu bound`, `blocking` and `asynchronous`, and gain some intuition about how they perform. Let's start by introducing each workload in more detail.

#### Flavors of Future

A `cpu bound` method will be using the cpu all the time. The following simulates such a method.

```
def cpuBoundMethod(v: Int): Int = {
    val start = System.currentTimeMillis()
    while ((System.currentTimeMillis() - start) < v) {}
    println(Thread.currentThread())
    v
}
```

We can create a Future with a cpu bound workload that will be delegated to another thread. The OS will schedule that thread's workload to run in one of our cores.

```
Future(cpuBoundMethod(time))
```

An `asynchronous` method will immediately return a future with a registered callback. Here we use a scheduler to simulate the callback. You should think that the scheduler does not belong to this program.

```
val scheduler: ScheduledExecutorService =
    Executors.newScheduledThreadPool(1)

def asyncMethod(v: Int)(implicit ec: ExecutionContext): Future[Int] = {
    val promise = Promise[Int]

    scheduler.schedule(new Runnable {
      override def run(): Unit = {
        promise.success(v)
      }
    }, v, TimeUnit.MILLISECONDS)

    promise.future.map { i =>
      println(Thread.currentThread)
      i
    }
}
```

A `blocking` method will put the current thread to sleep using the OS kernel calls. Here `Thread.sleep` happens to make that call.

```
def blockingMethod(v: Int): Int = {
    Thread.sleep(v)
    println(Thread.currentThread)
    v
}

Future(blockingMethod(time))
```

All futures are created with the `apply` method, which requires an `ExecutionContext`.

```
def apply[T](body: =>T)(implicit executor: ExecutionContext): Future[T]
```

In the examples that follow, we will schedule our futures to run in the Scala global `ExecutionContext` using it implicitly.

```
implicit val executionContext = scala.concurrent.ExecutionContext.Implicits.global
```

This ExecutionContext is a Java [Executor](https://docs.oracle.com/javase/tutorial/essential/concurrency/executors.html), with a thread pool equal to `Runtime.getRuntime().availableProcessors();`. In my case, `4`.

We will also be using the following method to wait for the result of a future in the main thread.

```
implicit class FutureOps[T](f: Future[T]) {
    def await(d: Duration = Duration.Inf): T = Await.result(f, d)
}
```

We can think of these workloads as solving the problems described in the first section as such:
1. Let other components execute when one is waiting on IO. (blocking, asynchronous)
2. Make small programs communicate to build larger ones. (blocking, asynchronous)
3. Use all the CPU's in our machines. (cpu bound)
4. Enable human-computer interaction (GUIs, terminal). (asynchronous)

#### CPU bound

Let's fill up the thread pool and consequentially our cores with actual work.

```
Future.sequence(List.fill(1000)(Future(cpuBoundMethod(1000)))).await()
```

On my machine, this produces the following output.

```
Thread[scala-execution-context-global-11,5,main]
Thread[scala-execution-context-global-13,5,main]
Thread[scala-execution-context-global-10,5,main]
Thread[scala-execution-context-global-12,5,main]
~1000ms PAUSE
Thread[scala-execution-context-global-10,5,main]
Thread[scala-execution-context-global-11,5,main]
Thread[scala-execution-context-global-12,5,main]
Thread[scala-execution-context-global-13,5,main]
~1000ms PAUSE
Thread[scala-execution-context-global-11,5,main]
Thread[scala-execution-context-global-10,5,main]
Thread[scala-execution-context-global-12,5,main]
Thread[scala-execution-context-global-13,5,main]
... continues until 1000 executions
```

And `htop` will consistently show my cpu usage as such:

```
1  [||||||||||||||||||||||||||||100.0%]
2  [||||||||||||||||||||||||||||100.0%]
3  [||||||||||||||||||||||||||||100.0%]
4  [||||||||||||||||||||||||||||100.0%]
```

We can see that our cores are filling up with work, but this does not mean our code *performs* better. Although this can be the case, what we may be trying to achieve using asynchronous code is *scalability*.

Creating a future and scheduling it has a cost, and there is a threshold for when it is worth to do this. After this threshold, the concept of amortization kicks in and we start to obtain scaling benefits.

Simplistically, if creating a future and scheduling it takes **10ms**, and our workload takes **10ms** to finish, then it will most likely perform worse. But if our workload takes **1000ms**, the cost of creating the future is amortized in the total cost, and it will scale.

Let's look at an example. The following runs the `cpuBoundMethod` function 4 times **sequentially** in the main thread.

```
val time: Int = ???
(1 to 4).foreach(cpuBoundMethod(time))
```

And using Future instead:

```
Future.sequence(List.fill(4)(Future(cpuBoundMethod(time)))).await()
```

My approximate simplistic results (in ms):

|           ms| Sequential | Future |
|-------------|------------|--------|
| time = 20   | 80         | 173    |
| time = 40   | 160        | 198    |
| time = 60   | 240        | 191    |
| time = 500  | 2000       | 594    |
| time = 1000 | 4000       | 1103   |

At 20ms, Future is too costly, At 40ms, Future begins to hit the threshold. At 500ms and 1000ms it scales better.

The above is actually a case of [Amdahl's law](https://en.wikipedia.org/wiki/Amdahl%27s_law). The serial part would be the overhead of creating Future in the main thread.

#### Blocking

The `blockingMethod` will produce output similar to the `cpuBoundMethod` method, but our cores will not be busy.

```
Future.sequence(List.fill(1000)(Future(blockingMethod(1000)))).await()
```

```
Thread[scala-execution-context-global-11,5,main]
Thread[scala-execution-context-global-10,5,main]
Thread[scala-execution-context-global-12,5,main]
Thread[scala-execution-context-global-13,5,main]
~1000ms PAUSE
Thread[scala-execution-context-global-11,5,main]
Thread[scala-execution-context-global-12,5,main]
Thread[scala-execution-context-global-10,5,main]
Thread[scala-execution-context-global-13,5,main]
... continues until 1000 executions
```

```
1  [||||                          8.9%]
2  [|||                           8.1%]
3  [|||||                        10.6%]
4  [||||                          7.3%]
```

Blocking calls will occupy the threads with no meaningful cpu work. Let's see what happens if we mix blocking and cpu bound workloads and run them on the same thread pool:

```
def rand = if (Random.nextInt(100) > 50) Future(cpuBoundMethod(1000)) else Future(blockingMethod(1000))
Future.sequence(List.fill(1000)(rand)).await()
```

Doing this will cause our application to arbitrarily use the cpu. At some point we might have this utilization:

```
1  [||||                          8.9%]
2  [|||                           8.1%]
3  [||||||||||||||||||||||||||||100.0%]
4  [||||                          7.3%]
```

As we can see, if we have heterogeneous workload of cpu intensive tasks and blocking, we are actually wasting resources and slowing down our application. A common solution to this problem has been to accept it as is and just schedule all tasks like we did.

But actually, the better solution is to spawn new threads and use those for blocking calls. We can make our application use 4 extra threads for blockingMethod calls:

```
val blockingContext = ExecutionContext.fromExecutor(Executors.newFixedThreadPool(4))
def rand = if (Random.nextInt(100) > 50) Future(cpuBoundMethod(1000)) else Future(blockingMethod(1000))(blockingContext)
Future.sequence(List.fill(1000)(rand)).await()
```

And we can see two different thread pools being used.

```
Thread[pool-1-thread-1,5,main] <- blocking calls
Thread[pool-1-thread-3,5,main]
Thread[pool-1-thread-2,5,main]
Thread[pool-1-thread-4,5,main]
Thread[scala-execution-context-global-11,5,main] <- cpu bound calls
Thread[scala-execution-context-global-14,5,main]
Thread[scala-execution-context-global-13,5,main]
Thread[scala-execution-context-global-17,5,main]
Thread[pool-1-thread-2,5,main]
Thread[pool-1-thread-1,5,main]
Thread[pool-1-thread-3,5,main]
Thread[pool-1-thread-4,5,main]
Thread[scala-execution-context-global-11,5,main]
Thread[scala-execution-context-global-14,5,main]
Thread[scala-execution-context-global-17,5,main]
Thread[scala-execution-context-global-13,5,main]
... continues until 1000 executions
```

`htop` will now print all cores at 100% utilization. The OS will manage the extra threads and put them to sleep, leaving our cpu threads mostly unaffected.

The downside is that spawning more threads than there are cores will eventually degrade performance on cpu bound tasks. Also, we are increasing the amount of context switches and memory usage per new thread. Finally, since system calls are expensive, this is also hurting our performance.

The only way around blocking is to go asynchronous, which we will look at next.

#### Asynchronous

Asynchronous calls are fast because a callback informs us of a result, so we are not wasting time waiting. We can understand this by looking at our `asyncMethod`. The scheduler is simulating a client side.

Let's take for example an arbitrary request. While blocking request may look like this:
```
time 1: Request value x
time 2: wait...
time 3: wait...
time 4: Obtain x
time 5: Request value y
time 6: wait...
time 7: wait...
time 8: Obtain y
```

An asynchronous Future will be more like this:
```
time 1: Request value x
time 2: Request value y
time 3:
time 4: Obtain x
time 5: Obtain y
```

What is the empty space in the asynchronous example? Any other work the cpu could be doing. Asynchronous code does not occupy time gaps with waiting.

*Note: The OS will put blocking threads to sleep, so the first example above is actually more efficient than shown here.*

Asynchronous futures are of a different species. All 1000 futures will be complete in ~1000ms.

```
Future.sequence(List.fill(1000)(asyncMethod(1000))).await()
```

```
... all previous executions
Thread[scala-execution-context-global-14,5,main]
Thread[scala-execution-context-global-14,5,main]
Thread[scala-execution-context-global-14,5,main]
Thread[scala-execution-context-global-14,5,main]
Thread[scala-execution-context-global-14,5,main]
Thread[scala-execution-context-global-13,5,main]
Thread[scala-execution-context-global-12,5,main]
End
```

These being so fast means there is less of a problem with scheduling them in an heterogeneous workload containing `cpuBoundMethod` calls, for example.

Also note that we have to be careful managing asynchronous futures, as we can go out of memory by creating large amounts of Future objects.

#### Thread Pools

So far we have used the default `scala.concurrent thread pool` and a `fixed thread pool` in our simple examples. We have scheduled cpu heavy futures in the first, and blocking futures in the second, in order to achieve more throughput.

The Java [Executors](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/Executors.html) class can spawn other executors, and we can convert them to an `ExecutionContext` as such:

```
ExecutionContext.fromExecutor(Executors.newFixedThreadPool(size))`
```

Let's describe all these executors in more detail:

##### `Executors.newSingleThreadExecutor`

This executor will run our futures sequentially in a single thread. It uses an unbounded queue for the futures awaiting execution, which means it can eventually throw `OutOfMemoryError`.

```
Future.sequence(List.fill(1000)(cpuBoundMethod(1000))).await()
```

```
Thread[scala-execution-context-global-1,5,main]
~1000ms PAUSE
Thread[scala-execution-context-global-1,5,main]
~1000ms PAUSE
Thread[scala-execution-context-global-1,5,main]
~1000ms PAUSE
Thread[scala-execution-context-global-1,5,main]
...
```

##### `Executors.newCachedThreadPool`

This executor will create a new thread each time it is required to run a future. Thread creation is unbounded and this means our application can go out of threads with a `OutOfMemoryError`. Idle threads will be released after 60 seconds by default.

Let's look at an example with fast computations where cpu time is equal to 10.

```
Future.sequence(List.fill(1000)(cpuBoundMethod(10))).await()
```

```
...
Thread[pool-2-thread-32,5,main]
Thread[pool-2-thread-16,5,main]
Thread[pool-2-thread-82,5,main]
Thread[pool-2-thread-79,5,main]
```

And now with our frequent example where cpu time is 1000.

```
Future.sequence(List.fill(1000)(cpuBoundMethod(1000))).await()
```

```
...
Thread[pool-2-thread-137,5,main]
Thread[pool-2-thread-84,5,main]
Thread[pool-2-thread-495,5,main]
Thread[pool-2-thread-126,5,main]
```

The pool increases from ~82 to ~495 threads from the fast to slower example.

##### `Executors.newFixedThreadPool(size)`

This executor will need to be passed a `size` and will always have that number of threads. The threads will obtain work from a shared unbounded queue. There exists contention on this queue, and the larger the size of the thread pool, the slower it will be for threads to obtain work. If our futures are fast, contention increases. If they are slow, contention decreases.

##### `Executors.newWorkStealingPool(parallelism)`

This executor is called fork/join and is similar to the fixed thread pool. It tries to reduce contention from threads obtaining work and tries to maintain the given `parallelism` level. This parallelism level is an attempt to keep `x` work running at the same time independent of threads, which can grow or shrink.

Contention is reduced by using multiple queues local to each thread, and also by making each thread be able to steal work from other threads queues when its queue is empty.

This executor was designed for fast recursive tasks. For long running futures, where contention is small, this executor may be unnecessary.

Refer to the [paper](http://gee.cs.oswego.edu/dl/papers/fj.pdf) for more details.

##### `scala.concurrent.ExecutionContext.Implicits.global`

This executor uses a fork join pool internally. It is actually a port from the Java implementation explained previously.

It provides a [feature](https://docs.scala-lang.org/overviews/core/futures.html) where we identify blocking calls, wrap them in a special method called `blocking` and the execution context will spawn extra threads to handle those calls.

#### Choosing thread pools

##### Blocking pool

When we looked at the `blocking` example, we decided to use two thread pools, one for blocking calls and one for cpu bound work.

The reason for doing this is that any blocking call escapes user mode and uses the OS capabilities for scheduling, which is slower since we need to use kernel calls. If we mix cpu bound work in user mode and much slower blocking code, we are hurting our application performance. The only solution to using blocking code somewhat efficiently is to manage it separately.

As a starting point, it should be easier to simply create a **newFixedThreadPool**. An upper bound for blocking threads must exist, or the application will run out of threads.

A **newCachedThreadPool** is also an option, but it requires extra code to verify that upper bounds on thread creation are met.

##### Asynchronous pool

These futures can usually be handled in a single thread. They are simply dispatching values so they should be very fast.

We can choose to schedule these in heterogeneous workloads but this can introduce turnaround problems, which means these faster futures may be blocked by long running futures. Separating these may make sense in order to improve throughput.

##### CPU bound pool

When doing cpu bounded work, it is best to use a thread pool size equal to the **number of cores**. After all, we are keeping all cores busy anyway so adding extra threads should only degrade performance with the extra context switching, kernel calls and thread memory usage.

The **scala.concurrent.ExecutionContext.Implicits.global** is probably the most widely used pool for this kind of workload. As mentioned before, this executor uses a fork join thread pool to manage the workloads. This type of executor is the best fit for highly asynchronous workloads but is less required when they are less asynchronous, because we don't have to deal with that much contention and stealing, which this pool tries to solve.

We can explain the level of concurrency for our workloads as such:

| ms        | Method        | Concurrency   |
|-----------|---------------|---------------|
| 0         | asyncMethod  | Highly asynchronous |
| 1         | Future(cpuBoundMethod(1)) | More asynchronous  |
| 5         | Future(cpuBoundMethod(5)) | Less asynchronous  |
| x       | cpuBoundMethod(x)   | Synchronous |

We can also visualize it as a function of the `cpuBoundMethod` time parameter. A `Future(cpuBoundMethod(x))` will be more asynchronous the closer it is to `1 ms`.

```
ms
|1------------------------------------------Inf|
```

#### Implicitly or explicitly

We looked at the `Future.apply` method definition before. It expects an implicit `ExecutionContext`, and this makes it tempting to always pass it implicitly in our applications.

For example, here we import it at top level and all our futures will pick it up:

```
object FutureApp extends App {
    import scala.concurrent.ExecutionContext.Implicits.global

    ...

    Future(blockingMethod(1))

    ...

    Future(cpuBoundMethod(1))
}
```

This may have consequences if that is not expected, for example when we want to use another executor for blocking.

##### Explicitly

To fix the above, we can obviously pass the blocking context explicitly and default the rest.

```
object FutureApp extends App {
    val defaultExecutor: ExecutionContext =  scala.concurrent.ExecutionContext.Implicits.global
    val blockingExecutor: ExecutionContext = ...

    ...

    Future(blockingMethod(1))(blockingExecutor)

    ...

    Future(cpuBoundMethod(1))
}
```

This is a micro management approach that can work when dealing with small code bases or teams.

##### Explicitly implicitly

We can improve on the above by typing our futures and thus marking them as blocking. This makes reutilization safer.

```
val defaultExecutor: ExecutionContext =  scala.concurrent.ExecutionContext.Implicits.global
val blockingExecutor: ExecutionContext = ...

case class BlockingContext(executionContext: ExecutionContext)

def markedBlocking(value: Int)(implicit blockingContext: BlockingContext) = {
    implicit val ec = blockingContext.executionContext
    Future(blockingMethod(value))
}

markedBlocking(10)(BlockingContext(blockingExecutor)) // compiles
markedBlocking(10)(defaultExecutor) // does not compile
```

The `markedBlocking` method would be public and used in place of our initial `blockingMethod`. This approach solves the boilerplate and risk of passing executors explicitly, documents our code and makes it safer for larger teams to understand where code requires a different execution strategy.

#### Interlude

So far, we should have gained some intuition on how `Future` and `ExecutionContext` work when interacting with three types of workloads.

We have seen how to manage `blocking`, which is a special case where we [delegate control to the OS](http://pages.cs.wisc.edu/~remzi/OSTEP/cpu-mechanisms.pdf).

We have also seen how an `asynchronous` method interacts with a future. Our asynchronous method is actually a special case where no computation is involved in our program, often referred to as asynchronous IO.

`cpu bound` remains as the most frequent workload a common application will probably have. We have shown [here](#cpu-bound-pool), [here](#cpu-bound) and [here](#thread-pools) how changing the running time and thus level of concurrency can impact our choices.

In a `Future(post)` we will be looking at the performance considerations of `Future` itself, and how it differs from current alternatives.
