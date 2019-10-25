---
layout: default
title: ZIO, Http4s, Auth, Codecs and zio-test
description: Examples of use of ZIO with the http4s library, illustrating http4s authentication, custom codes and testing with zio-test
---


# ZIO - Http4s Part4 - Custom Codecs

Codecs are used to convert Http request and response bodies to and from the
things that are meaningful to your program.

Http4s comes with support for json via circe and some other lower-level codecs.
But it is by no means a complete set for all requirements.

This example shows how to construct a custom codec for use with http4s and Zio. In this
case the data type is *Person* and we're going to encode it in xml.

Note, there is quite a lot of information plus available examples on http4s Encoders and
Decoders on the http4s web site and in the source code. I used the circe
json encoders and decoders as models.


```scala
// Simple data type used for encoding and decoding
case class Person(name: String, age: Int)

object Person {
  val donald = Person("Donald Trump", 73)
  val joe = Person("Joe Biden", 76)
}
```

### Encoding

We will use the standard scala xml library. To make the code general we use an XmlWriter typeclass that produces a scala.xml.Node
```scala
  trait XmlWriter[A] {
    def write(a: A): Node
  }
```

For Person the implementation is very simple, using the scala xml api
```scala
  implicit val personXmlWriter: XmlWriter[Person] = { p =>
    <Person>
      <name>{p.name}</name>
      <age>{p.age}</age>
    </Person>
  }
```

Next we need to create the implicit encoder
```scala
 implicit def xmlEntityEncoder[F[_] : Monad, X](implicit writer: XmlWriter[X]): EntityEncoder[F, X] =
    EntityEncoder
      .stringEncoder[F]
      .contramap { x: X =>
        val node = writer.write(x)
        val s = new PrettyPrinter(80, 2).format(node) // just because it's easier to debug
        s
      }
      .withContentType(`Content-Type`(MediaType.application.xml))
```
Breaking this down, we use an implicit XmlWriter. This encodes the content as a string - we've decided to pretty print the output
to aid in tracing and debugging. The Encoder is also responsible for adding a content-type to the message.

### Decoding
Again we use scala xml - here for the example a non-particularly subtle procedure. Our parser can produce an error message in this case
```scala
  trait XmlParser[A] {
    def parse(node: Node): Either[String, A]
  }
  
  implicit val personXmlParser: XmlParser[Person] = { node =>
    try {
      val name = (node \ "name").head.text
      val age = (node \ "age").head.text.toInt
      Right(Person(name, age))
    } catch {
      case e : Exception => Left("parse error in xml")
    }
  }  
```
Purely for testing purposes I added an additional utility method
```scala
  def parseIO[T](s: String)(implicit xmlParser: XmlParser[T]): Task[T] =
    IO.effect {
      val xml = XML.loadString(s)
      xmlParser.parse(xml) match {
        case Left(err) => throw new Exception(err)
        case Right(t) => t
      }
    }
```

### Using the encoders and decoders

Ensure your decoder and encoder are in scope
```scala
import zhx.encoding.Encoders._

object Hello3Service {

  private val dsl = Http4sDsl[Task]
  import dsl._

  val service = HttpRoutes.of[Task] {
    case GET -> Root => Ok("hello3!")
    case GET -> Root / "president" => Ok(Person.donald) // uses implicit encoder
    case req @ POST -> Root / "ageOf" =>
      req.decode[Person] { m => Ok(m.age.toString)}
  }.orNotFound
}
```
The actual encoding and decoding in the service is all done by implicits. So
```scala
  case GET -> Root / "president" => Ok(Person.donald) // uses implicit encoder
```
will automatically create a 200 response with the body of *MediaType.application.xml* and the appropriate data.

and
```scala
    case req @ POST -> Root / "ageOf" =>
      req.decode[Person] { m => Ok(m.age.toString)}
```
will expect to decode an xml body of type person.

## Testing
The service can be tested directly.
```scala
object TestHello3Service extends DefaultRunnableSpec(
  suite("routes suite")(
    testM("president returns donald") {
      for{
        response <- Hello3Service.service.run(Request[Task](Method.GET, uri"/president"))
        body <- response.body.compile.toVector.map(x => x.map(_.toChar).mkString(""))
        parsed <- parseIO(body)
      }yield assert(parsed, equalTo(Person.donald))
    },
    testM("joe is 76") {
      val rq = Request[Task](Method.POST, uri"/ageOf")
        .withEntity(Person.joe)
      for {
        response <- Hello3Service.service.run(rq)
        body <- response.body.compile.toVector.map(x => x.map(_.toChar).mkString(""))
      } yield assert(body, equalTo("76"))
    }

  ))
```

## Putting it all together - Authentication plus Codecs

So to wrap it up, the final service uses both Authentication and custom codecs:

```scala
class Hello4Service[R <: Authenticator] {

  type AuthenticatorTask[T] = RIO[R, T]
  private val dsl = Http4sDsl[AuthenticatorTask]
  import dsl._

  val service = AuthedRoutes.of[AuthToken, AuthenticatorTask] {
    case GET -> Root as authToken => Ok("hello4!")
    case GET -> Root / "president" as authToken => Ok(Person.donald) // uses implicit encoder
    case AuthedRequest(authToken, req @ POST -> Root / "ageOf") =>
      req.decode[Person] { m => Ok(m.age.toString)}
  }

}
```

Tested with:
```scala
object TestHello4Service extends DefaultRunnableSpec(
  suite("routes suite")(
    testM("president returns donald") {
      val req1 = Request[withMiddleware.AppTask](Method.GET, uri"/president")
      val req = AuthenticationHeaders.addAuthentication(req1, "tim", "friend")
      val io = (for{
        response <- hello4Service.run(req)
        body <- response.body.compile.toVector.map(x => x.map(_.toChar).mkString(""))
        parsed <- parseIO(body)
      }yield parsed)
        .provide(new Authenticator{ override val authenticatorService = Authenticator.friendlyAuthenticator})
      assertM(io, equalTo(Person.donald))
    },
    testM("joe is 76") {
      val req1 = Request[withMiddleware.AppTask](Method.POST, uri"/ageOf")
      val req = AuthenticationHeaders.addAuthentication(req1, "tim", "friend")
        .withEntity(Person.joe)
      val io = (for{
        response <- hello4Service.run(req)
        body <- response.body.compile.toVector.map(x => x.map(_.toChar).mkString(""))
      }yield body)
        .provide(new Authenticator{ override val authenticatorService = Authenticator.friendlyAuthenticator})
      assertM(io, equalTo("76"))
    }

  ))


object MoreMiddlewares {
  val hello4Service1 = new Hello4Service[Authenticator]
  val hello4Service = Router[withMiddleware.AppTask](
    "" -> withMiddleware.authenticationMiddleware(hello4Service1.service))
    .orNotFound

}
```




Note that due to the rapid rate of development of Zio prior to the 1.0 release and external factors,
both code samples and data may be out of date by the time you read this post.
