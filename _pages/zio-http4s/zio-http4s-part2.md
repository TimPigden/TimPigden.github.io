---
layout: default
title: ZIO, Http4s, Auth, Codecs and zio-test
description: Examples of use of ZIO with the http4s library, illustrating http4s authentication, custom codes and testing with zio-test
---

## Testing with Http4s Client and ZIO

In the last post we created a simple ZIO/Http4s server. But in addition to server we are already using, Http4s provides a client.
We can use it to test our web server or indeed other web services. Note that full testing of an http4s server at this level
may not be strictly necessary - we have already tested the individual service end points and so are basically testing http4s ourselves.
However, it is also a useful template for testing external services.

First we create a convenience method to take care of creating the BlazeClientBuilder:
```scala
object ClientTest {

  def testClientM[R](fClient: Client[Task] => Task[TestResult])
  : Task[TestResult] =
    ZIO.runtime[Any].flatMap { implicit rts =>
      val exec = rts.Platform.executor.asEC
      BlazeClientBuilder[Task](exec).resource.use { client =>
        fClient(client)
      }
    }
}
```
We need access to the ZIO.runtime for 2 reasons:

Firstly, we want to use the runtime's execution context (val exec above)
rather than global. While not particularly important within this test, it would be more relevant
if the Client was constructed in main.

Secondly, it requires an implicit ConcurrentEffect.

Having got the Client object we run the function fClient which returns a Task[TestResult]

Note in the above code, the line
```scala
BlazeClientBuilder[Task](exec).resource.use { client =>
```
This is BlazeClientBuilder's mechanism for executing code with the client. the .resource.use makes
use of the cats.effect Resource to ensure that the client is safely opened and subsequently released, freeing associated resources.

The test itself is another DefaultRunnableSpec
```scala
object TestHello1 extends DefaultRunnableSpec(

  suite("routes suite")(
    testM("test get") {
      ClientTest.testClientM { client =>
        val req = Request[Task](Method.GET, uri"http://localhost:8080/")
        assertM(client.status(req), equalTo(Status.Ok))
      }
    }
  )
)
```
Here the testM passes the actual test - an anonymous function - to testClientM to execute.

Note that the server needs to be running to execute this test.

## Wrapping the Client in a Service

There is a minor problem with the above. If we actually have lots of tests to run, we keep re-creating our client for every test.
This may not be particularly expensive, but we can eliminate it with  a construct that allows us to implement BeforeAll and AfterAll style
actions on our test suite - which could be more important for other resources such as database connections.

Managed provides a mechanism for safe resource usage within ZIO. See the [relevant page](https://zio.dev/docs/datatypes/datatypes_managed) of the Zio documentation.
This is essentially the same technique as used in cats.effect.Resource which we used in the first version.
So cats.effect and zio are both doing the same thing but with different underlying mechanisms. What we want to do here is turn the
cats.effect Resource into a Managed so that we can use it more fluently within our Zio test environment.

However, ZIO RC18 also introduces Layers - a mechanism for providing stuff in the environment in a convenient manner. See zio docs and also [my layers blog](https://timpigden.github.io/_pages/zlayer/Examples.html)

### Wrapping Resource
The library [interop-cats](https://github.com/zio/interop-cats) provides, among other things, translation between Resource and Managed.
The code below shows how it can be used:
```scala
def clientManaged = {
    val zioManaged: ZIO[Any, Throwable, ZManaged[Any, Throwable, Client[Task]]] = ZIO.runtime[Any].map { rts =>
      val exec = rts.platform.executor.asEC

      implicit def rr = rts

      catz.catsIOResourceSyntax(BlazeClientBuilder[Task](exec).resource).toManaged
    }
    // for our test we need a ZManaged, but right now we've got a ZIO of a ZManaged. To deal with
    // that we create a Managed of the ZIO and then flatten it
    val zm = zioManaged.toManaged_ // toManaged_ provides an empty release of the rescoure
    zm.flatten
  }
```

### Creating the ZLayer

We will adopt the common zio module pattern for our layer. 
```scala
package zhx.client
import org.http4s.client.Client
import zhx.client.HClient.HClient
import zio._
object HClient {

  type HClient = Has[Service]

  trait Service {
    def client: Client[Task]
  }

  case class SimpleClient(client: Client[Task]) extends Service
}

package object hClient {
  def client = ZIO.access[HClient](_.get.client)
}
```
This wraps the http4s Client in an HClient module with convenience method and a SimpleClient implementation as a case class.

Next we need to wrap our managed resource in a layer
```scala
def clientLive:ZLayer[Any, Throwable, HClient] = ZLayer.fromManaged(clientManaged.map(x => SimpleClient(x)))
```

### Using the resource
Finally we need to use the resource. Our modified test looks like this:
```scala
override def spec = suite("routes suite")(
    testM("test get") {
      for {
        client <- hClient.client
        req = Request[Task](Method.GET, uri"http://localhost:8080/")
        asserted <- assertM(client.status(req))(equalTo(Status.Ok))
      } yield asserted
    }
  ).provideCustomLayerShared(ClientTest.clientLive).mapError(TestFailure.fail) @@ sequential
```

Note that we're adding the resource with provideCustomLayerShared. This adds the same client to all the tests - ok, we've only the one in this case, but in general we would expect more tests and this means our client is only created once. The sequential annotation ensures that the tests run sequentially (since I'm not sure whether underlying http4s Client is going to work thread-safely).
