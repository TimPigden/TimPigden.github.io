---
layout: default
permalink: "/zio-http4s-part1.md/"
---

# ZIO and Http4s with auth, codecs and zio-test

There are a couple of sample projects already in existence showing you how to use zio with http4s as a server. So why another?

What this adds to the mix is an http4s Authentication/Authorization example and an example of custom encoding and decoding.  And tests are written using the zio-test module.

For a detailed description of the various libraries you should head on over to the relevant pages. However, if, like me, you sometimes struggle a bit to get everything working together, then this may help.

I will not attempt to explain or argue for either of these libraries, but briefly, ZIO is the latest in a series of scala effects libraries which includes cats.IO and Monix. Http4s is a popular typelevel web framework based on the Blaze server (there is also a client).

The github project contains 4 sets of services that can be treated as a progression from simplest to most complex:

* Hello1 is a very basis "hello world" web service
* Hello2 adds an authentication layer.
* Hello3 is expands 1 with a custom encoding and decoding of an xml object, using scala xml
* Hello combines 2 & 3

Today's blog covers Service1 and testing using the new zio-test framework.

# Hello!

Service1 is a simple service comprising a single endpoint at the root with a GET return text "hello!"

ZIO provides us with Task[T], which is defined as ZIO[Any, Throwable, T]. See the documentation on ZIO interop packages and in particular interop.cats for a general explanation of this

```scala
object Hello1Service {

  private val dsl = Http4sDsl[Task]
  import dsl._

  val service = HttpRoutes.of[Task] {
    case GET -> Root => Ok("hello!")
  }.orNotFound
}
```

The http4s dsl import is required to provide various implicits of type [Task]

For the service we create a single route returning Ok("hello!). The orNotFound serves to provide a page 404 in the event of an incorrect url path.

So HelloService.service provides the HttpRoutes. This is then encapsulated within the server, which looks like this:

```scala

object Hello1 extends App {

  val server = ZIO.runtime[Environment]
    .flatMap {
      implicit rts =>
        BlazeServerBuilder[Task]
          .bindHttp(8080, "localhost")
          .withHttpApp(Hello1Service.service)
          .serve
          .compile
          .drain
    }
    
   def run(args: List[String]) =
    server.fold(_ => 1, _ => 0)

}

```

In this case App is the ZIO App which provides access to the ZIO runtime. The run method runs the server, with fold return 1 in the event of an error and 0 for normal termination.

To construct BlazeSErverBuilder[Task] we need an in-scope ConcurrentEffect. This is provided by the ZIO runtime - which is simply and effectful call to get the ZIO runtime object from the environment.  
Note, within the BlazeServerBuilder we are binding to localhost:8080

The .withHttpApp(Hello1Service.service) is defining the service we run.

That's it. Get the code and run Hello1, try it out in the browser with http://localhost:8080

# Testing the service

## Testing TestHello1Service directly

You can unit test the service without starting up the Blaze server and web browser.
The following code illustrates this
```scala
package zhx.servers

import org.http4s._
import zio._
import zio.interop.catz._
import zio.test._
import zio.test.Assertion._

object TestHello1Service extends DefaultRunnableSpec (
  suite("routes suite")(
    testM("root request returns Ok") {
      for {
        response <- Hello1Service.service.run(Request[Task](Method.GET, uri"/"))
      } yield assert(response.status, equalTo(Status.Ok))
    },
    testM("root request returns Ok, using assertM insteat") {
      assertM(Hello1Service.service.run(Request[Task](Method.GET, uri"/")).map(_.status),
      equalTo(Status.Ok))
    },
    testM("root request returns NotFound") {
      assertM(Hello1Service.service.run(Request[Task](Method.GET, uri"/a")).map(_.status),
        equalTo(Status.NotFound))
    },
    testM("root request body returns hello!") {
      val io = for{
        response <- Hello1Service.service.run(Request[Task](Method.GET, uri"/"))
        body <- response.body.compile.toVector.map(x => x.map(_.toChar).mkString(""))
      }yield body
      assertM(io, equalTo("hello!"))
    }

  ))
```

For those of you familiar with Specs2 or ScalaTest, this will look broadly familiar in shape. However, it uses neither of these libraries, instead using
DefaultRunnableSpec from the zio-test package (check out the build.sbt file to see how that is defined, also this link XXXXX)

The zio test module has no dependencies other than on ZIO itself. It contains assert and assertM statements, plus scala-check like generators.
Note that you need to combine successive asserts with && for them both to run and report properly - unlike scalaTest for example.

Each test executes the route directly, with Hello1Service.service.run, providing an appropriate url.  
This returns a Task[Response[Task] The status is extracted from the response with .map and compared to the expected answer using equalTo.

The first example uses a for comprehension. The value response is therefore a Response[Task]. This is checked with assert.

The alternative version assertM expects a ZIO[...] as its first argument. It is particularly useful for one-line checks. The for comprehension is better suited to multi-part operations.

Our first test requests a GET of "/", which, as expected, returns a status of OK. Url /a gives us a NotFound since there was no match. The 3rd test
is slightly more elaborate. First, we do our request then we extract the body of the response - this is standard http4s. First the body is "compiled" which
unpacks the stream of bytes that are turned into a Vector[Byte] with toVector. These are then turned into chars and finally our vector of chars is
turned into a string with mkString.
the val io is thus a Task[String] and this is compared with assertM.

## Testing with curl

If you start up the Hello1 server you can test with curl. Both the following work:

Try the following if you have curl installed
```
curl localhost:8080/  # returns hello!
curl localhost:8080/a # returns NotFound
```

## Testing with Http4s Client

In addition to the server we are already using, Http4s provides a client.
