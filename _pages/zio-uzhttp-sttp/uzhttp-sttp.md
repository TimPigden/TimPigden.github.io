---
layout: default
title: uzhttp + sttp 
description: Using uzhttp with sttp for http and web sockets
---
# ZIO-based Http Server and Client using uzhttp and sttp

**NB this is for zio RC18-2. Currently waiting for libraries to catch up to upgrade to RC19**

A few months ago I wrote a blog on [using http4s from zio](../zio-http4s/intro.md) - which I've just updated to zio RC18-2

In that blog I went into quite a lot of detail looking at authentication and at encoding, because both had proved time-consuming to get right.

Recently, a new zio-based been launched by the authors of the [polynote](https://polynote.org/) - the netflix-originated analytics notebook. (uzhttp)[(https://github.com/polynote/uzhttp)] is a micro-http server - it's very lightweight and comes with no bells and whistles, though it does support websockets. It's used in polynote apparently. But it does come with a health warning - and may be too micro for most people.

Nevertheless, I thought I'd compare it with the http4s solution to see how easy it was to work with. And this blog shows the first fruits.

All source code is on my [github](https://github.com/TimPigden/zio-http4s-examples). It's one sbt project with subprojects and code for this is in the uzsttp repository

Thanks to the zio regulars who gave me tips in this work plus Jeremy Smith from polynote.org. Please comment or drop me a line if there are any errors or omissions or opportunities to improve the blog or the code.

## Client
Http4s provides both a server and a client. And we need both a server and a client for our testing. So rather than re-use http4s I thought I'd try (sttp)[https://github.com/softwaremill/sttp]. Unlike uzhttp, sttp is a battled hardend scala http client solution that's been around quite a while. It has versions for a variety of backend and is pretty comprehensive. Importantly, one of those backends is a Netty-based zio implementation and this is what we shall be using.

I'm not going to go into detail of sttp - it's very well documented and was really easy to use, so just go and look at the website

## Testing
For the http4s testing, it was possible to test the "routes" without starting up a full server or using the client. This is currently not possible with uzhttp because of the package design where much is private (as far as I can tell). So instead, the first example will be the full-blown test that starts a server and a client.

## Hello!

So here's the hello server app. There's a single http endpoint
```scala
object Hello1Routes {
  val routes: PartialFunction[Request, IO[HTTPError, Response]] = {
    case req if (req.uri.getPath === "/") && (req.method === Method.GET) =>
      IO.succeed(Response.plain("OK"))
  }
}
```

So that's pretty simple - it's a PartialFunction taking a uzhttp.Request and returning an IO[HTTPError, Response]. It doesn't do a lot!.

Here's the test
```scala
object HelloServerTest extends DefaultRunnableSpec {

  def hasRoot = testM("service has root") {
    for {
      _ <- serverUp
      response <- SttpClient.send(basicRequest.get(uri"http://localhost:8080/"))
    } yield assert(response.code)(equalTo(StatusCode.Ok))
  }

  def hasBody = testM("service has body") {
    for {
      _ <- serverUp
      response <- SttpClient.send(basicRequest.get(uri"http://localhost:8080/"))
    } yield {
      assert(response.body)(equalTo(Right("OK")))
    }
  }

  override def spec = suite("all tests")(
    testHello1
  )

  val testHello1 = suite("test hello1 with sttp client")(
    hasRoot,
    hasBody,
  ).provideCustomLayerShared(AsyncHttpClientZioBackend.layer() ++ serverLayer(Hello1Routes.routes)).mapError(TestFailure.fail)

}
```

Two tests, each of which checks the server is running and then uses SttpClient to send a basic request and return the response. 

Because the test is running across the wire, you won't see any uzhttp code or types here. We just need to know that the server is started and is running.

The actual work is done in the ZIO layers that have been created - see Zio documentation and [my previous blog]()../zlayer/Examples.md) for more about Zio layers. This article assumes you have familiarity with the concept.

So we're creating 2 layers - the first is the client. This puts SttpClient.send .. into our context. It's created with a direct call to ```AsyncHttpClientZioBackend.layer()``` 
and that's it. Like I said - really simple.

The server is a bit more work. We call the function serverLayer with the partial function we listed earlier. The code looks like this:
```scala
  def serverLayer(handler: PartialFunction[Request, IO[HTTPError, Response]]) = ZLayer.fromManaged(
    Server.builder(new InetSocketAddress("127.0.0.1", 8080))
  .handleSome(handler)
  .serve
  )
```

The uzhttp4s Server.builder has a builder pattern to create and start the builder. It returns a ZIO Managed which we can simply wrap up in the ZLayer.fromManaged to give the layer.

To check server is started and running
```scala
  type UZServer = Has[Server]

  def serverUp = ZIO.access[UZServer](_.get).map{_.awaitUp}
```

## Encoding
For the previous blog we looked at a custom encoding and we will do the same here. The encoding is into xml using the scala.xml library. We're using the same Person type as last time:
```scala
case class Person(name: String, age: Int)

object Person {
  val donald = Person("Donald Trump", 73)
  val joe = Person("Joe Biden", 76)
}```

We obviously need to get and post data, so our XmlRoutes is a bit more involved:
```scala
object XmlRoutes {
  val routes: PartialFunction[Request, IO[HTTPError, Response]] = {
    case req if (req.uri.getPath.startsWith("/president")) && (req.method === Method.GET) =>
      IO.succeed(
        writeXmlBody(Person.donald)
      )
    case req if (req.uri.getPath.startsWith("/whatIsMyName")) && (req.method === Method.POST) =>
      extractXmlBody[Person](req).map{ p =>
        Response.plain(p.name)
      }
  }
}
```

### GET
So for the GET we're calling writeXmlBody which is in Encoders
```scala
object Encoders {
  // .. parser stuff omitted
  trait XmlWriter[A] {
    def write(a: A): Node
  }
  implicit val personXmlWriter: XmlWriter[Person] = { p =>
    <Person>
      <name>{p.name}</name>
      <age>{p.age}</age>
    </Person>
  }

  def writeXmlString[T](t: T)(implicit xmlWriter: XmlWriter[T]) = {
    // extravagently spaced pretty version for ease of debugging 
    val pretty = new PrettyPrinter(80, 2)
    pretty.format(xmlWriter.write(t))
  }

  def xmlResponse(body: String, status: Status = Status.Ok, headers: List[(String, String)] = Nil, charset: Charset = StandardCharsets.UTF_8): Response =
    Response.const(body.getBytes(charset), status, contentType = s"application/xml; charset=${charset.name()}", headers = headers)

  def writeXmlBody[T](t: T)(implicit xmlWriter: XmlWriter[Tuzhttp]) = {
    xmlResponse(writeXmlString(t))
  }
}

```
So we've defined an typeclass XmlWriter[A]. In the real world I will be using a magnolia-based XmlWriter but here we have a a scala.xml dsl-based one. 

The code is pretty obvious, we pretty-print the xml (for testing) and then use the uzhttp Response.const function to actually create a response. There are other functions to create Responses in uzhttp. They are not documented - you will have to go and look at the source code - but take comfort - it's nice and easy to read and quite short. This code is a direct rip-off the Response.html method but with different content type

### POST

The POST is a bit more complicated. Our particular method wants to extract a Person from Xml then it just returns the name in ```text/plain``` Here are the missing bits of Encoders:
```scala
object Encoders {

  case class ParseError(msg: String) extends Throwable(msg)

  trait XmlParser[A] {
    def parse(node: Node): Task[A]
  }
  implicit val personXmlParser: XmlParser[Person] = { node =>
    try {
      val name = (node \ "name").head.text
      val age = (node \ "age").head.text.toInt
      IO.succeed(Person(name, age))
    } catch {
      case e : Exception => IO.fail(ParseError(e.getMessage()))
    }
  }

  def parseXmlString[T](s: String)(implicit xmlParser: XmlParser[T]): IO[Throwable, T] =
    for {
      validXml <- Task(XML.loadString(s))
      parsed <- xmlParser.parse(validXml)
    } yield parsed

  def extractStringBody(req: Request): IO[HTTPError, String] =
    req.body match {  def serverUp = ZIO.access[UZServer](_.get).map{_.awaitUp}

        case Some(value) => 
          value.run(ZSink.utf8DecodeChunk)            
        case None        => ZIO.fail(BadRequest("Missing body"))
    }

  def extractXmlBody[T](req: Request)(implicit xmlParser: XmlParser[T]): IO[HTTPError, T] =
    for {
      s <- extractStringBody(req)
      _ = println(s"extracted string body $s")
      t <- parseXmlString(s)(xmlParser).mapError(e => BadRequest(e.getMessage))
    } yield t

}

```

So again we've got a typeclass XmlParser and implemented in full for Person - though in the real world again I would have used magnolia.

So it's all pretty obvious apart from this bit
```scala
  def extractStringBody(req: Request): IO[HTTPError, String] =
    req.body match {
        case Some(value) => 
          value.run(ZSink.utf8DecodeChunk)            
        case None        => ZIO.fail(BadRequest("Missing body"))
    }
```
uzhttp uses ZStream (Zio streams) to manage the body data. **req.body** is of type ```Option[StreamChunk[HTTPError, Byte]]```. So if we've got one, we need to grab the chunked byte stream and turn it into text. In this case we run it into a standard ZStream sink that does the job for us and returns an IO of a String. There are similar methods for byte array and so on.

Finally our test:
```scala
  def hasDonald = testM("we have a president") {
    for {
      _ <- serverUp
      response <- SttpClient.send(basicRequest.get(uri"http://localhost:8080/president"))
      body = response.body
      goodBody <- body match {
        case Left(errs) => IO.fail(new Throwable(s"bad body $errs"))
        case Right(bdy) => parseXmlString[Person](bdy)
      }
    } yield assert(goodBody)(equalTo(donald))
  }

  def isJoe = testM("joe's name comes back") {
    for {
      _ <- serverUp
      response <- SttpClient.send(basicRequest.post(uri"http://localhost:8080/whatIsMyName")
      .body(writeXmlString(joe)))
    } yield assert(response.body)(equalTo(Right(joe.name)))
  }

  def badBodyJoe = testM("badRequest") {
    for {
      _ <- serverUp
      response <- SttpClient.send(basicRequest.post(uri"http://localhost:8080/whatIsMyName")
        .body("joe was the vp"))
    } yield assert(response.code)(equalTo(StatusCode.BadRequest))
  }


  override def spec = suite("all tests")(
    hasDonald,
    isJoe,
    badBodyJoe
  ).provideCustomLayerShared(AsyncHttpClientZioBackend.layer() ++ serverLayer(XmlRoutes.routes)).mapError(TestFailure.fail)

```
Obviously we need our XmlWriter and XmlParser to create and processed the string bodies that are used with sttp.

## Authorization and Authentication

uzhttp makes no provision for authorisation or authentication. As the author's say - it's intended to be used behind a reverse proxy. However, in my world I'm happy to let the outer wall do authentication, but my authorization is application-specific so it makes sense to do something about that. 

The Authorizer is going to be held in a ZLayer using Zio module pattern. Here are the main definitions
```scala
object Authorizer {

  type Authorizer = Has[Service]

  case class AuthInfo(status: String)

  object AuthInfo {
    val empty = AuthInfo("Dont care")
  }

  type AccessToken = String

  val Authorization = "Authorization"

  def getAuthorization(req: Request): RIO[Authorizer, AuthInfo] =
    req.headers.get(Authorization) match {
      case None => IO.fail(Unauthorized(""))
      case Some(s) => authorizer.authorize(s)
    }

  trait Service {
    def authorize(token: AccessToken): Task[AuthInfo]
  }
```

Essentially the Authorizer takes an access token which has been provided by the external environment. It will then validate this to return AuthInfo - in this case just a wrapped string but in reality will be something more complex (and yes in real world probably tokens will expire and so on)

Next we provide a dummy Authorizer
```
  val friendlyAuthorizer: Service = { token =>
    token match {
      case "friend" => IO.succeed(AuthInfo("Vetted"))
      case "acquaintance" => IO.succeed(AuthInfo("Dodgy"))
      case _ => IO.fail(Unauthorized("sorry, but no entry"))
    }
  }

  val friendlyAuthorizerLive = ZLayer.succeed(friendlyAuthorizer)
```
This recognises just 2 possible tokens and provides 2 levels of AuthInfo. For tokens that have no Authorization we provide Unauthorized status code (401)

Our AuthorizedRoutes contains the actual http routes
```
object AuthorizedRoutes {
  val routes: PartialFunction[(Request, AuthInfo), IO[HTTPError, Response]] = {
    case (req, auth) if (req.uri.getPath === "/") && (req.method === Method.GET) =>
      if (auth.status === "Vetted") IO.succeed(Response.plain("OK"))
      else IO.fail(Forbidden("go get permission"))
  }
}
```

This is similar to the ones we've had before, but with a critical different - it is 
```
: PartialFunction[(Request, AuthInfo), IO[HTTPError, Response]]
```
Previously we had 
```
PartialFunction[Request, IO[HTTPError, Response]]
```
and this is what the uzhttp is expecting. So we've got to deal with this in some way. My Authorizer.Service is in a Layer, so I'm going to need to use that to provide a transformed PartialFunction. This is achieved with the following rather messy code:
```scala
  def authorized(needsAuthority: PartialFunction[(Request, AuthInfo), IO[HTTPError, Response]]):
  ZIO[Authorizer, HTTPError, PartialFunction[Request, IO[HTTPError, Response]]] =
    ZIO.access[Authorizer](_.get).map { aut =>
      new PartialFunction[Request, IO[HTTPError, Response]] {
        override def isDefinedAt(x: Request): Boolean = needsAuthority.isDefinedAt((x, AuthInfo.empty))
        override def apply(x: Request): IO[HTTPError, Response] =
          (for {
            authInfo <- getAuthorization(x).provideLayer(ZLayer.succeed(aut))
            applied <- needsAuthority.apply((x, authInfo))
          } yield applied)
          .mapError { th =>
            th match {
              case herr: HTTPError => herr
              case th =>   Unauthorized(th.getMessage)
            }
          }
      }
    }
```
So I grab my authorization and create a new PartialFunction in the required type. But unfortunately, PartialFunction.isDefinedAt returns a Boolean. We can't use the actual AuthInfo from the Authorizer to write our new PartialFunction.isDefinedAt - because that would need a ZIO here - so isDefinedAt is not able to properly check the AuthThoken (correction or fixes on this point welcome).

Going to our test, we have quite a few tests:
```scala
object AuthServerTest extends DefaultRunnableSpec {

  override def spec = suite("all tests")(
    testAuth
  )

  val noAuthentication = testM("root request with no authentication returns Unauthorized") {
    for {
      _ <- serverUp
      response <- SttpClient.send(basicRequest.get(uri"http://localhost:8080/"))
    } yield assert(response.code)(equalTo(StatusCode.Unauthorized))
  }

  val noAuthorization = testM("root request with authentication but no authorization returns") {
    for {
      _ <- serverUp
      response <- SttpClient.send(
        basicRequest.get(uri"http://localhost:8080/")
          .header(Authorizer.Authorization, "anybody")
      )

    } yield assert(response.code)(equalTo(StatusCode.Unauthorized))
  }

  val insufficientAuthorization = testM("root request with authentication and low level authorisation") {
    for {
      _ <- serverUp
      response <- SttpClient.send(
        basicRequest.get(uri"http://localhost:8080/")
          .header(Authorizer.Authorization, "acquaintance")
      )

    } yield assert(response.code)(equalTo(StatusCode.Forbidden))
  }

  val sufficientAuthorization = testM("root request with authentication and high level authorisation") {
    for {
      _ <- serverUp
      response <- SttpClient.send(
        basicRequest.get(uri"http://localhost:8080/")
          .header(Authorizer.Authorization, "friend")
      )
    } yield assert(response.code)(equalTo(StatusCode.Ok))
  }

  val notFoundTrumpsNoAuthentication = testM("no auth, wrong page gives not found") {
    for {
      _ <- serverUp
      response <- SttpClient.send(
        basicRequest.get(uri"http://localhost:8080/a")
      )
    } yield assert(response.code)(equalTo(StatusCode.NotFound))
  }

  val notFoundTrumpsAuthentication = testM("good auth, wrong page gives not found") {
    for {
      _ <- serverUp
      response <- SttpClient.send(
        basicRequest.get(uri"http://localhost:8080/a")
          .header(Authorizer.Authorization, "friend")
      )
    } yield assert(response.code)(equalTo(StatusCode.NotFound))
  }

  val testAuth = suite("test authorization sttp client")(
    noAuthentication,
    noAuthorization,
    insufficientAuthorization,
    sufficientAuthorization,
    notFoundTrumpsNoAuthentication,
    notFoundTrumpsAuthentication,
  ).provideCustomLayerShared(AsyncHttpClientZioBackend.layer() ++
    ((Blocking.live ++ Clock.live ++ Authorizer.friendlyAuthorizerLive) >>> authLayer(AuthorizedRoutes.routes))).mapError(TestFailure.fail)

}
```

This is mainly to check that I get the right errors back for the various failure cases. the most interesting bit is the final couple of lines:
```scala
.provideCustomLayerShared(AsyncHttpClientZioBackend.layer() ++
    ((Blocking.live ++ Clock.live ++ Authorizer.friendlyAuthorizerLive) >>> authLayer(AuthorizedRoutes.routes))).mapError(TestFailure.fail)
```
The sttp layer is the same, but the server layer is rather more complicated as we need to add the Authorizer into it. Our utiltiies has the missing authLayer
```
  def serverLayerM[R](handlerM: RIO[R, PartialFunction[Request, IO[HTTPError, Response]]]) =
    ZLayer.fromManaged {
      val zm = handlerM.map { handler =>
        Server.builder(new InetSocketAddress("127.0.0.1", 8080))
          .handleSome(handler)
          .serve
      }
      ZManaged.unwrap(zm)
    }

  def authLayer(handler: PartialFunction[(Request, AuthInfo), IO[HTTPError, Response]]):
    ZLayer[Blocking with Clock with Authorizer, Throwable, Has[Server]] =
    serverLayerM[Authorizer](Authorizer.authorized(handler))
```
It's slightly more complicated due to the fact that we have a ZIO of a Managed to deal with. We unwrap that first with ZManaged.unwrap before applying ZLayer.fromManaged

# Can we do better?

Personally, I find having long lists of PartialFunction case matches not particularly satisfactory. uzhttp authors say they had no intention of making a DSL but can we make our code more fluent.

One possibility was suggested to me on the zio-users discord channe. Instead of combining partial functions, we can make it more "zio-like" with the following:
```
  type HRequest = Has[Request]

  type EndPoint[R <: HRequest] = ZIO[R, Option[HTTPError], Response]
```

Much like our partial function, it takes a request and returns a response. But what's the Option[HTTPError] about? Essentially, what it does is allows us to give us 3 return possibilities:
- the Response, if our EndPoint matches the request
- An error of Some(error) if there's something wrong with the Request
- An "error" of None - if the EndPoint doesn't match the response

Now we can write individual EndPoints like this:
```scala
  val president = uriMethod(startsWith("president"), Method.GET).as {
    writeXmlBody(Person.donald)
  }

  val contender = uriMethod(endsWith("contender"), Method.GET).as {
    writeXmlBody(Person.joe)
  }

  val whatIsMyName = for {
    _ <- uriMethod(endsWith(NonEmptyList.of("whatIsMyName")), Method.POST)
    person <- parsedXmlBody[Person]
  } yield Response.plain(person.name)
```
where the helper methods are things like:
```scala
  def uriMethod(pMatch: Seq[String] => Boolean, expectedMethod: Method): ZIO[HRequest, Option[HTTPError], Unit] = {
    for {
      pth <- uri
      mtd <- method
      matched <- if (pMatch(pth) && (mtd === expectedMethod))
        IO.unit else IO.fail(None)
    } yield matched
  }

```

So that's individual EndPoints. How do we chain them together - what is the equivalent to "orElse". So looking through the zio RC18-2 I couldn't see a really slick way of doing this. Asking on the Discord channel, essentially elicited the response, there's nothing there yet, ok, we've just done something. So in zio RC19 or 1.0.0 or a current (post about April 20th) SNAPSHOT you will be able to do this:
```
val routes = president orElseOptional  contender orElseOptional whatIsMyName
```
But for the impatient, I've got the following:
```
  def combineRoutes[R <: HRequest](h: EndPoint[R], t: EndPoint[R]*): EndPoint[R] =
    t.foldLeft(h)((acc, it) =>
      acc catchSome { case None => it }
    )
```
giving:
```scala
  val routes = combineRoutes(president, contender, whatIsMyName)
```
To my mind this is nicer and tidier than the equivalent use of PartialFunction. Our auth endpoints become:
```scala
  val authorized: EndPoint[HRequest with Auth] =
    for {
      _ <- uriMethod(endsWith("authorized"), Method.GET)
      _ <- authStatus("Vetted")
    } yield Response.plain("OK")
// where
  def authStatus(s: String): ZIO[Auth, Option[HTTPError], Unit] =
    for {
      stat <- auth.status
      res <- if (stat === s) IO.unit
      else IO.fail(Some(Forbidden("go get permission")))
    } yield res
```

Sorting out the layers to provide Auth is a bit more straight-forward than adding it into the PartialFunction. You still have to do a bit of work though:
```scala
  def authHandler(p: EndPoint[HRequest with Auth]): ZIO[Authorizer, HTTPError, Request => IO[HTTPError, Response]] =
    ZIO.access[Authorizer](_.get).map { aut => { req: Request =>
      (for {
        authInfo <- getAuthorization(req).provideLayer(ZLayer.succeed(aut))
        res <- orNotFound(p).provideLayer(ZLayer.succeed(req) ++ ZLayer.succeed(authInfo))
      } yield res).mapError { th =>
        th match {
          case herr: HTTPError => herr
          case th => Unauthorized(th.getMessage)
        }
      }}
    }
```

# Websockets
Finally, we come to websockets. Both sttp and uzhttp implement websockets, so how are they used? The following is a very brief example. uzhttp uses websockets via zio zstream. Apologies, this was hastily assembled to give an idea or what you could do rather than make any claim to best practice. In particular I am no expert on ZStream!

This is the endPoint code:
```scala
  def agePerson(text: String): IO[HTTPError, Text] =
    parseXmlString[Person](text).map { person =>
      Text(writeXmlString(older(person)), true)
    }.mapError(e => BadRequest(e.getMessage))


  val agePeople: EndPoint[HRequest] =
    for {
      req <- webSocket.mapError(e => Some(e))
      _ <- uriMethod(endsWith("wsIn"), Method.GET)
      streamOut = Stream.flatten(req.frames.mapM(handleWebsocketFrame(agePerson))).unTake
      response <- Response.websocket(req, streamOut).mapError(e => Some(e))
    } yield response
```
Supporting code:
```scala
  def webSocket: ZIO[HRequest, HTTPError, WebsocketRequest] =
    for {
      r <- request
      ws <- r match {
        case wr: WebsocketRequest => IO.succeed(wr)
        case x => IO.fail(BadRequest("not a websocket"))
      }
    } yield ws

   def handleWebsocketFrame(textHandler: String => IO[HTTPError, Frame])
                          (frame: Frame): UIO[Stream[HTTPError, Take[Nothing, Frame]]] = frame match {
    case frame@Binary(data, _)       => UIO.succeed(Stream.empty)
    case frame@Text(data, _)         => textHandler(data)
        .either.map {
          case Left(err) => Stream.fail(err)
          case Right(f) => Stream(Take.Value(f))
    }
    case frame@Continuation(data, _) => UIO.succeed(Stream.empty)
    case Ping => UIO(Stream(Take.Value(Pong)))
    case Pong => UIO(Stream.empty)
    case Close => UIO(Stream(Take.Value(Close), Take.End))
  }
```
So first to explain the EndPOint code:

The purpose of the EndPoint is to take an incoming Person and add 1 to the age, returning it as the websocket response.

We call the webSocket function that takes the request and checks it is a valid websocket request (as opposed to an ordinary http one). Incoming data is on a stream. We provide a handler - in this case agePerson than takes a string and outputs a Text (a websocket Frame with text body). 

handleWebsocketFrame - which only works for text frames in this example, deals with the stream stuff. Note that we're using Stream[HTTPError, Take[Nothing, Frame]]. There are also curious responses to Ping and Pong - which are ways that websockets maintain that there's still someone on the line.

Test code here just does a single message, but sttp is well-documented and you should be able to figure out how to make a more extensive app.
```scala
  def sendPerson(person: Person, ws: WebSocket[Task]) = {
    println(s"sending person $person")
    ws.send(WebSocketFrame.text(writeXmlString(person)))
  }

  def next(ws: WebSocket[Task]): Task[Option[Person]] =
    for {
      et <- ws.receiveText()
      personOpt <- et match {
        case Right(t) => parseXmlString(t).map(Some(_))
        case _ => IO.succeed(None)
      }
    } yield personOpt

  val peopleAge = testM("test age people"){

    for {
      _ <- serverUp
      response <- SttpClient.openWebsocket(basicRequest.get(uri"ws://localhost:8080/wsIn"))
      _ = println(s"response is $response")
      ws = response.result
      sent <- sendPerson(joe, ws)
      joeOlder <- next(ws)
    } yield assert(joeOlder)(equalTo(Some(older(joe))))
```

That's it. Full source code avaialble at: [github](https://github.com/TimPigden/zio-http4s-examples)























