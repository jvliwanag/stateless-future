Immutable Future
================

`immutable-future` is a set of DSL for asynchronous programming, in the pure functional favor.

## Usage

    import scala.concurrent.duration._
    import scala.util.control.Exception.Catcher
    import com.qifun.immutableFuture.Future
    
    val executor = java.util.concurrent.Executors.newSingleThreadScheduledExecutor
    
    // Manually implements an immutable future, which is the asynchronous version of `Thread.sleep()`
    def asyncSleep(duration: Duration) = new Future[Unit] {
      import scala.util.control.TailCalls._
      def onComplete(handler: Unit => TailRec[Unit])(implicit catcher: Catcher[TailRec[Unit]]) = {
        executor.schedule(new Runnable {
          def run() {
            handler().result
          }
        }, duration.length, duration.unit)
        done()
      }
    }
    
    // Without the keyword `new`, you have the magic version of `Future` constructor,
    // which enables the magic postfix `await`.
    val sleep10seconds = Future {
      var i = 0
      while (i < 10) {
        println(s"I have sleeped $i times.")
        // The magic postfix `await` invokes asynchronous method like normal `Thread.sleep()`,
        // and does not block any thread.
        asyncSleep(1.seconds).await
        i += 1
      }
      i
    }
    
    // When `sleep10seconds` is running, it could report failture to this catcher
    implicit def catcher: Catcher[Unit] = {
      case e: Exception => {
        println("An exception occured when I was sleeping: " + e.getMessage)
      }
    }
    
    // An immutable future instance is lazy, only evaluating when you query it.
    println("Before the evaluation of the immutable future `sleep10seconds`.")
    for (total <- sleep10seconds) {
      println("After the evaluation of the immutable future `sleep10seconds`.")
      println(s"I sleeped $total times in total.")
      executor.shutdown()
    }


Run it and you will see the output:

    Before evaluation of the immutable future `sleep10seconds`.
    I have sleeped 0 times.
    I have sleeped 1 times.
    I have sleeped 2 times.
    I have sleeped 3 times.
    I have sleeped 4 times.
    I have sleeped 5 times.
    I have sleeped 6 times.
    I have sleeped 7 times.
    I have sleeped 8 times.
    I have sleeped 9 times.
    After evaluation of the immutable future `sleep10seconds`.
    I sleeped 10 times in total.

## Further Information

There are two sorts of API to use an immutable future, the for-comprehensions style API and "A-Normal Form" style API.

### For-Comprehensions

The for-comprehensions style API for `immutable-future` is like the [for-comprehensions for scala.concurrent.Future](http://docs.scala-lang.org/overviews/core/futures.html#functional_composition_and_forcomprehensions). 

    for (total <- sleep10seconds) {
      println("After evaluation of the immutable future `sleep10seconds`")
      println(s"I sleeped $total times in total.")
      executor.shutdown()
    }

A notable difference between the two for-comprehensions implementation is the required implicit parameter. `scala.concurrent.Future` requires an `ExecutionContext`, while immutable future requires a `Catcher`.

    import scala.util.control.Exception.Catcher
    implicit def catcher: Catcher[Unit] = {
      case e: Exception => {
        println("An exception occured when I was sleeping: " + e.getMessage)
      }
    }

### A-Normal Form

"A-Normal Form" style API for immutable futures is like the pending proposal [Async](http://docs.scala-lang.org/sips/pending/async.html).

    val sleep10seconds = Future {
      var i = 0
      while (i < 10) {
        println(s"I have sleeped $i times")
        // The magic postfix `await` invokes asynchronous method like normal `Thread.sleep()`,
        // and does not block any thread.
        asyncSleep(1.seconds).await
        i += 1
      }
      i
    }

The `Future` functions for immutable futures correspond to `async` method in `Async`, and the `await` postfixes to immutable futures corresponds to `await` method in `Async`.

## Design

Regardless of the familiar veneers between immutable futures and `scala.concurrent.Future`, I have made some different designed choices on immutable futures.

### Immutability

Immutable futures are stateless, they will never store result values or exceptions. Instead, the immutable futures evaluate lazily, and they do the same work for every time you invoke `foreach` or `onComplete`. The behavior of immutable futures is more like monads in Haskell than futures in Java.

Also, there is no `isComplete` method in immutable futures. As a result, the users of immutable futures are forced not to share futures between threads, not to check the states in futures. They have to care about control flows instead of threads, and build the control flows by defining immutable futures.

### Threading-free Model

There are too many threading models and implimentations in the Java/Scala world, `java.util.concurrent.Executor`, `scala.concurrent.ExecutionContext`, `javax.swing.SwingUtilities.invokeLater`, `java.util.Timer`, ... It is very hard to communicate between threading models. When a developer is working with multiple threading models, he must very careful to pass messages between threading models, or he have to maintain bulks of `synchronized` methods to properly deal with shared variables.

Why does he need multiple threading models? Because the libraries that he uses depend on different threading modes. For example, you must update Swing components in the Swing's UI thread, you must specify `java.util.concurrent.ExecutionService`s for `java.nio.channels.CompletionHandler`, and, you must specify `scala.concurrent.ExecutionContext`s for `scala.concurrent.Future` and `scala.async.Async`. Oops!

Think about somebody who uses Swing to develop a text editor software. He wants to create a state machine to update UI. He have heard the cool `scala.async`, then he uses the cool "A-Normal Form" expression in `async` to build the state machine that updates UI, and he types `import scala.concurrent.ExecutionContext.Implicits._` to suppress the compiler errors. Everything looks pretty, except the software always crashes.

Fortunately, `immutable-future` depends on none of these threading model, and cooperates with all of these threading models. If the poor guy tries `immutable-future`, replacing `async { }` to `Future { }`, deleting the `import scala.concurrent.ExecutionContext.Implicits._`, he will find that everything looks pretty like before, and does not crash any more. That's why threading-free model is important.

### Exception Handling

There were two `Future` implementations in Scala standard library, `scala.actors.Future` and `scala.concurrent.Future`. `scala.actors.Future`s are taken from [Akka](http://akka.io/), which do not designed to handling exceptions, since Akka's exceptions handlers are associated with the threads running actors.

Unlike Akka futures, `scala.concurrent.Future`s are designed to handle exceptions. But, unfortunately, `scala.concurrent.Future`s provide too many mechanisms to handle an exception. For example:

    import scala.concurrent.Await
    import scala.concurrent.ExecutionContext
    import scala.concurrent.duration.Duration
    import scala.util.control.Exception.Catcher
    import scala.concurrent.forkjoin.ForkJoinPool
    val catcher1: Catcher[Unit] = { case e: Exception => println("catcher1") }
    val catcher2: Catcher[Unit] = {
      case e: java.io.IOException => println("catcher2")
      case other: Exception => throw new RuntimeException(other)
    }
    val catcher3: Catcher[Unit] = {
      case e: java.io.IOException => println("catcher4")
      case other: Exception => throw new RuntimeException(other)
    }
    val catcher4: Catcher[Unit] = { case e: Exception => println("catcher4") }
    val catcher5: Catcher[Unit] = { case e: Exception => println("catcher5") }
    val catcher6: Catcher[Unit] = { case e: Exception => println("catcher6") }
    val catcher7: Catcher[Unit] = { case e: Exception => println("catcher7") }
    def future1 = scala.concurrent.future { 1 }(ExecutionContext.fromExecutor(new ForkJoinPool(), catcher1))
    def future2 = scala.concurrent.Future.failed(new Exception)
    val composedFuture = future1.flatMap { _ => future2 }(ExecutionContext.fromExecutor(new ForkJoinPool(), catcher2))
    composedFuture.onFailure(catcher3)(ExecutionContext.fromExecutor(new ForkJoinPool(), catcher4))
    composedFuture.onFailure(catcher5)(ExecutionContext.fromExecutor(new ForkJoinPool(), catcher6))
    try { Await.result(composedFuture, Duration.Inf) } catch { case e if catcher7.isDefinedAt(e) => catcher7(e) }

Is any sane developer able to tell which catchers will receive the exceptions?



### Tail Call Optimization
