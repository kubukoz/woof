# Woof

[![Maven Central](https://maven-badges.herokuapp.com/maven-central/org.legogroup/woof-core_3/badge.svg)](https://maven-badges.herokuapp.com/maven-central/org.legogroup/woof-core_3)
[![Scala CI](https://github.com/LEGO/woof/actions/workflows/scala.yml/badge.svg?branch=main)](https://github.com/LEGO/woof/actions/workflows/scala.yml)

A **pure** _(in both senses of the word!)_ **Scala 3** logging library with **no runtime reflection**.

![](dog-svgrepo-com.svg)

## Highlights

* Pure **Scala 3** library
* Made with _Cats Effect_
* Macro based (_no runtime reflection_)
  * ~~Can be built for _scala.js_ in the future!~~
  * Cross-built for `Scala.js`
* Configured with plain Scala code

## Installation

> build.sbt

```scala
libraryDependencies ++= Seq(
  "org.legogroup" %% "woof-core"   % "$VERSION",
  "org.legogroup" %% "woof-slf4j"  % "$VERSION", // only if you need to use Woof via slf4j
  "org.legogroup" %% "woof-http4s" % "$VERSION", // only if you need to add correlation IDs in http4s 
)
```

## Example 

```scala
import cats.effect.IO
import org.legogroup.woof.{given, *}

val consoleOutput: Output[IO] = new Output[IO]:
  def output(str: String)      = IO.delay(println(str))
  def outputError(str: String) = output(str) // MDOC ignores stderr

given Filter = Filter.everything
given Printer = NoColorPrinter()

def program(using Logger[IO]): IO[Unit] = 
  for
    _ <- Logger[IO].debug("This is some debug")
    _ <- Logger[IO].info("HEY!")
    _ <- Logger[IO].warn("I'm warning you")
    _ <- Logger[IO].error("I give up")
  yield ()

val main: IO[Unit] = 
  for
    given Logger[IO]  <- DefaultLogger.makeIo(consoleOutput)
    _                 <- program
  yield ()
```

and running it yields:

```scala
import cats.effect.unsafe.implicits.global
main.unsafeRunSync()
// 2022-04-21 08:47:11 [DEBUG] repl.MdocSession$.App: This is some debug (.:27)
// 2022-04-21 08:47:11 [INFO ] repl.MdocSession$.App: HEY! (.:28)
// 2022-04-21 08:47:11 [WARN ] repl.MdocSession$.App: I'm warning you (.:29)
// 2022-04-21 08:47:11 [ERROR] repl.MdocSession$.App: I give up (.:30)
```


We can also re-use the program and add context to our logger:

```scala
import Logger.*
val mainWithContext: IO[Unit] = 
  for
    given Logger[IO]  <- DefaultLogger.makeIo(consoleOutput)
    _                 <- program.withLogContext("trace-id", "4d334544-6462-43fa-b0b1-12846f871573")
    _                 <- Logger[IO].info("Now the context is gone")
  yield ()
```

And running with context yields:

```scala
mainWithContext.unsafeRunSync()
// 2022-04-21 08:47:11 [DEBUG] trace-id=4d334544-6462-43fa-b0b1-12846f871573 repl.MdocSession$.App: This is some debug (.:27)
// 2022-04-21 08:47:11 [INFO ] trace-id=4d334544-6462-43fa-b0b1-12846f871573 repl.MdocSession$.App: HEY! (.:28)
// 2022-04-21 08:47:11 [WARN ] trace-id=4d334544-6462-43fa-b0b1-12846f871573 repl.MdocSession$.App: I'm warning you (.:29)
// 2022-04-21 08:47:11 [ERROR] trace-id=4d334544-6462-43fa-b0b1-12846f871573 repl.MdocSession$.App: I give up (.:30)
// 2022-04-21 08:47:11 [INFO ] repl.MdocSession$.App: Now the context is gone (.:61)
```

# Can I use SLF4J?

Yes, you can. I don't think you should (for new projects), but you can use it for interop with existing SLF4J programs! Note, however, that not everything can be implemented perfectly against the
`SLF4J` API, e.g. the filtering functionality in `woof` is much more flexible and thus does not map directly to, e.g., `isDebugEnabled`.

> NOTE: This is about implementing the `SLF4J` API for `woof`, **not** about sending `woof` logs INTO existing SLF4J implementations

Consider this program which logs using the `SLF4J` API

```scala
import org.slf4j.LoggerFactory
def programWithSlf4j: IO[Unit] = 
  for
    slf4jLogger <- IO.delay(LoggerFactory.getLogger(this.getClass))
    _           <- IO.delay(slf4jLogger.info("Hello from SLF4j!"))
    _           <- IO.delay(slf4jLogger.warn("This is not the pure woof."))
  yield ()
```

To use this program with woof

1. add `woof-slf4j` as a dependency to our program
1. instantiate a `woof.Logger[F[_]]` as per usual
1. register the _woof logger_ to the static log binder to allow the slf4j `LoggerFactory` to find it.

> Note that any logs that happen before registration are lost!

```scala
import org.legogroup.woof.slf4j.*
val mainSlf4j: IO[Unit] = 
  for
    woofLogger  <- DefaultLogger.makeIo(consoleOutput)
    _           <- woofLogger.registerSlf4j
    _           <- programWithSlf4j
  yield ()
```

and running it:

```scala
mainSlf4j.unsafeRunSync()
// 2022-04-21 08:47:11 [INFO ] repl.MdocSession$App: Hello from SLF4j! (MdocSession$App.scala:81)
// 2022-04-21 08:47:11 [WARN ] repl.MdocSession$App: This is not the pure woof. (MdocSession$App.scala:82)
```
## Limitations of SLF4J bindings

Currently, markers do nothing. You can get the same behaviour easily with context when using the direct `woof` api with filters and printers.

# Can I use __http4s__?

Yes you can. If you want to see internal logs from `http4s`, use the `SLF4J` module from above. If you want to use the context capabilities in `woof`, there's a module for adding correlation IDs to each request with a simple middleware.

> NOTE: The correlation ID is also added to the response header when using this middleware

Consider the following `http4s` route:

```scala
import org.http4s.{HttpRoutes, Response}
import cats.data.{Kleisli, OptionT}
import cats.syntax.functor.given

def routes(using Logger[IO]): HttpRoutes[IO] =
  Kleisli(request =>
    OptionT
      .liftF(Logger[IO].info("I got a request with trace id! :D"))
      .as(Response[IO]()),
  )
```

We create a tracing middleware from the above routes and call the resulting
route with an empty request.

```scala
import org.http4s.Request
import org.legogroup.woof.http4s.CorrelationIdMiddleware
import cats.syntax.option.given

val mainHttp4s: IO[Unit] = 
  for
    given Logger[IO]  <- DefaultLogger.makeIo(consoleOutput)
    maybeResponse     <- CorrelationIdMiddleware.middleware[IO]()(routes).run(Request[IO]()).value
    responseHeaders   =  maybeResponse.map(_.headers).orEmpty
    _                 <- Logger[IO].info(s"Got response headers: $responseHeaders")
  yield ()
```

Finally, running it, we see that the correlation ID is added to the log message inside the routes (transparently), and that 
the correlation ID is also returned in the header of the response.

> NOTE: The correlation ID is _not_ present outside the routes, i.e. we have scoped it only to the service part of our code.

```scala
mainHttp4s.unsafeRunSync()
// 2022-04-21 08:47:12 [INFO ] X-Trace-Id=f9780ca5-2e41-445e-96c7-335822d2143b repl.MdocSession$.App: I got a request with trace id! :D (.:121)
// 2022-04-21 08:47:12 [INFO ] repl.MdocSession$.App: Got response headers: Headers(X-Trace-Id: f9780ca5-2e41-445e-96c7-335822d2143b) (.:142)
```

## Structured Logging

Structured logging is useful when your logs are collected and inspected by a monitoring system. Having a well structured log output can save you 
hours of reg-ex'ing your way towards the root cause of a burning issue.

`Woof` supports printing as `Json`:

```scala
import Logger.*
val contextAsJson: IO[Unit] = 
  given Printer = JsonPrinter()
  for
    given Logger[IO]  <- DefaultLogger.makeIo(consoleOutput)
    _                 <- program.withLogContext("foo", "42").withLogContext("bar", "1337")
    _                 <- Logger[IO].info("Now the context is gone")
  yield ()
```

And running with context yields:

```scala
contextAsJson.unsafeRunSync()
// {"level":"Debug","epochMillis":1650523632092,"timeStamp":"2022-04-21T06:47:12Z","enclosingClass":"repl.MdocSession$.App","message":"This is some debug","context":{"bar":"1337","foo":"42"}}
// {"level":"Info","epochMillis":1650523632096,"timeStamp":"2022-04-21T06:47:12Z","enclosingClass":"repl.MdocSession$.App","message":"HEY!","context":{"bar":"1337","foo":"42"}}
// {"level":"Warn","epochMillis":1650523632096,"timeStamp":"2022-04-21T06:47:12Z","enclosingClass":"repl.MdocSession$.App","message":"I'm warning you","context":{"bar":"1337","foo":"42"}}
// {"level":"Error","epochMillis":1650523632096,"timeStamp":"2022-04-21T06:47:12Z","enclosingClass":"repl.MdocSession$.App","message":"I give up","context":{"bar":"1337","foo":"42"}}
// {"level":"Info","epochMillis":1650523632097,"timeStamp":"2022-04-21T06:47:12Z","enclosingClass":"repl.MdocSession$.App","message":"Now the context is gone","context":{}}
```


> We are considering if we should support matching different printers with different outputs: Maybe you want human readable logs for standard out and structured logging for your monitoring tools. However, this will be a breaking change.