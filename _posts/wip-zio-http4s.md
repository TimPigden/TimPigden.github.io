# ZIO and Http4s with auth, codecs and zio-test

There are a couple of sample projects already in existence showing you how to use zio with http4s as a server. So why another?

What this adds to the mix is an http4s Authentication/Authorization example and an example of custom encoding and decoding.  And tests are written using the zio-test module.

For a detailed description of the various libraries you should head on over to the relevant pages. However, if, like me, you sometimes struggle a bit to get everything working together, then this may help.

I will not attempt to explain or argue for either of these libraries, but briefly, ZIO is the latest in a series of scala effects libraries which includes cats.IO and Monix. Http4s is a popular typelevel web framework based on the Blaze server (there is also a client).

The github project contains 4 sets of services that can be treated as a progression from simplest to most complex:

* Service1 is a very basis "hello world" web service
* Service2 adds an auhtentication layer.
* Service3 is expands 1 with a custom encoding and decoding of an xml object, using scala xml
* Service4 combines 2 & 3

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

The server itself invokes ZIO runtime flatmapping over the BlazeServerBuilder. Check the http4s docs for a description of the various actions within the builder. For our purposes its a standard pattern, the interesting bits beig the fact that it is defined with Task and that we are running on localhost 8080.

The .withHttpApp(Hello1Service.service) is defining the service we run.

That's it. Get the code and run Hello1, try it out in the browser with http://localhost:8080

## Testing the service
You can also test the service without starting up the Blaze server and web browser.
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

For those of you familiar with Specs2 or ScalaTest, this will look broadly familiar in shape. However, it uses neithr of these libraries, instead using
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

### Testing with curl

Try the following if you have curl installed
```
curl localhost:8080/  # returns hello!
curl localhost:8080/a # returns NotFound
```

# Hello2

Obviously "hello1" is not particularly interesting by iteself. In our second example we will show how to authenticate our request and pass an authentication object to the service.

## The Service
First we create Hello2Service which looks like this:
```scala
class Hello2Service[R <: Authenticator] {

  type AuthenticatorTask[T] = RIO[R, T]
  private val dsl = Http4sDsl[AuthenticatorTask]
  import dsl._

  val service = AuthedRoutes.of[AuthToken, AuthenticatorTask] {
    case GET -> Root as authToken => Ok(s"hello! ${authToken.tok}")
  }

}
```

The first thing is that we have a class that takes a type parameter. In this case R is a ZIO Environment. Please see the ZIO documentation and blogs for an explanation of environment. In this case
our environment need.

## Authenticator
s to contain a single thing - an Authenticator

```scala
object Authenticator {

  case class AuthToken(tok: String)

  trait AuthenticationError extends Throwable

  val authenticationError: AuthenticationError = new AuthenticationError {
    override def getMessage: String = "Authentication Error"
  }

  trait Service {
    def authenticate(userName: String, password: String): Task[AuthToken]
  }

  val friendlyAuthenticator: Service = { (userName, password) =>
    password match {
      case "friend" => IO.succeed(AuthToken(userName)) // rather trivial implementation but does allow us to inject variety
      case _ => IO.fail(authenticationError)
    }
  }
}

trait Authenticator { def authenticatorService: Authenticator.Service }
```

Essentially, an Authenticator is something that can provide an Authenticator.Service. The service takes a userName and password and returns Task[AuthToken]

So in our authentication process, we are going to extract username and password from the request (in this case from the request header) and check them. If they are ok, we get an AuthToken. If not, we will
expect a Task failure. In this case it will be of type AuthenticationError, our custom return type, which needs to extend Throwable to conform to the definition of Task.

In the sample you will also see friendlyAuthenticator - a sample service which will authenticate anyone with "friend" as their password. Our example AuthToken is simply the string of the username. Please be more dilligent in your own authentication process!

Note that the definition of Authenticator with Authenticator.Service is a standard pattern for dealing with your own environment variables, that you will see elsewhere, both in the Zio codebase (if you look) or examples and blogs.

## Headers
Our Authenticator.Service requires username and password, which are to come from the request. So how do we get these?
```scala
trait AuthenticationHeaders[R <: Authenticator] {
  type AuthHTask[T] = RIO[R, T]

  private def unauthenticated = IO.succeed(Left(new Exception("bad format authentication")))

  def getToken(req: Request[AuthHTask]) : AuthHTask[Either[Throwable, AuthToken]] = {
    val userNamePasswordOpt: Option[Array[String]] =
      for {
        auth <- req.headers.get(Authorization).map(_.value)
        asSplit = auth.split(" ")
        if asSplit.size == 2
      } yield asSplit
    val tok = userNamePasswordOpt.map { asSplit =>
      val res1 = for {
        authentic <- authenticator.authenticatorService
        tok  <- authentic.authenticate(asSplit(0), asSplit(1))
      } yield tok
      res1.either
    }
    tok.getOrElse(unauthenticated)
  }
}
```
In this trivial implementation, they are simply grabbed from a header, using
the Authorization tag from http4s and splitting the string on space. Don't
do this at home please!.

Note that our example returns an IO.succeed[Either[Throwable, AuthToken]] - this is because http4s middleware expects unauthorized as a Left[Throwable]
rather than a task failure (that's just the way it works, not our decision).

## Back to Service

Returning to Hello2Service, the next couple of lines are
```scala
  type AuthenticatorTask[T] = RIO[R, T]
  private val dsl = Http4sDsl[AuthenticatorTask]
  import dsl._
```
Compared with Hello1, you can see that our Http4sDsl is now typed with AuthenticatorTask instead of simply task.  
Essentially we have provided Task with an environment that contains our Authenticator.

Finally in Hello2Service we have
```scala
  val service = AuthedRoutes.of[AuthToken, AuthenticatorTask] {
    case GET -> Root as authToken => Ok(s"hello! ${authToken.tok}")
  }
```

So instead of HttpRoutes.of[Task] from Hello1 we now have AuthedRoutes.of[AuthToken, AuthenticatorTask]

AuthedRoutes is a standard part of the http4s authentication middleware and you should refer to the relevant documentation for a more complete description.
At the moment the critical thing is that the case GET line has changed. We now extract an authToken as part of the pattern match. The authToken.tok contains the username
and our answer will now be hello! <username>

So now we have defined our Authenticator, we can extract headers and we have a Hello2Service that will use it.  Next up, we need to create the middleware layer.

## middleware

One further piece of the puzzle is the middleware. This is a wrapper that takes the request, extracts the header and supplies the authenticcated token that
allows us to call our AuthedRoutes.

```scala
trait AuthenticationMiddleware {

  type AppEnvironment <: Authenticator
  type AppTask[A] = RIO[AppEnvironment, A]

  val dsl: Http4sDsl[AppTask] = Http4sDsl[AppTask]
  import dsl._

  val authenticationHeaders = new AuthenticationHeaders[AppEnvironment] {}

  def authUser: Kleisli[AppTask, Request[AppTask], Either[String, AuthToken]] = {
    Kleisli({ request =>
      authenticationHeaders.getToken(request).map { e => {
        e.left.map (_.toString)
      }}
    }
    )
  }

  val onFailure: AuthedRoutes[String, AppTask] = Kleisli(req => OptionT.liftF {
    Forbidden(req.authInfo)
  })

  val authenticationMiddleware: AuthMiddleware[AppTask, AuthToken] = AuthMiddleware(authUser, onFailure)
}
```

First up, we define our AppEnvironment as one that contains an Authenticator. This is the the R of the ZIO[R, E, T] and is required to extract
the authentication context. Remember, we used a trivial Authenticator, but in real life it might well be something that talks to an external
authentication service such as Google or OpenAuth.

Next we construct a dsl to provide appropriately typed http4s implicits.

We create a headers object to extract our headers

The following code may well look unfamiliar:
```

  def authUser: Kleisli[AppTask, Request[AppTask], Either[String, AuthToken]] = {
    Kleisli({ request =>
      authenticationHeaders.getToken(request).map { e => {
        e.left.map (_.toString)
      }}
    }
    )
  }
```
A Kleisli is from cats where the api docs describe it as "Represents a function A => F[B]". If that's part of your programming bread and butter, then
fine, but to those of you who only got as far as Functional Programming 101, it may seem a little scary.
And I'm not going to explain it here. But don't worry, the code fragment above works just fine and can be readily adapted to your own authentication (or other)
middleware requirements.

The onFailure function serves to deal with authentication failures and is just telling the system to respond with a Forbidden message.

Finally, we create an AuthMiddleware which combines the authUser and onFailure functions.

## Hello2

So we have got most of the moving parts. But how do we link them all together?

```scala

object Hello2 extends App with AuthenticationMiddleware {

  type AppEnvironment = Clock with Console with Authenticator with Blocking

  val hello2Service = new Hello2Service[AppEnvironment] {}

  val authenticatedService = authenticationMiddleware(hello2Service.service)

  val secApp = Router[AppTask](
    "" -> authenticatedService
  ).orNotFound

  val server1 = ZIO.runtime[AppEnvironment]
    .flatMap {
      implicit rts =>
        BlazeServerBuilder[AppTask]
          .bindHttp(8080, "localhost")
          .withHttpApp(secApp)
          .serve
          .compile
          .drain
    }

  val server = server1
    .provideSome[Environment] { base =>
      new Clock with Console with Blocking with Authenticator {
        override val clock: Clock.Service[Any] = base.clock
        override val console: Console.Service[Any] = base.console
        override val blocking: Blocking.Service[Any] = base.blocking

        override def authenticatorService: Authenticator.Service = Authenticator.friendlyAuthenticator
      }
    }
    
  def run(args: List[String]): ZIO[Environment, Nothing, Int] =
    server.foldM(err => putStrLn(s"execution failed with $err") *> ZIO.succeed(1), _ => ZIO.succeed(0))


}
```

This has suddenly got rather more complicated.

Our App is extended with AuthenticationMiddleware, which means our types line up.

Next we need to define an Environment that includes Authenticator.
These are the elements of the standard ZIO environment we need, plus Authenticator.

We create the new hello2Service instance of the right type parameterisation.

Next we wrap our hello2Service in our authenticationMiddleware. The result of this operation is not an HttpRoutes - in fact intellij says it's
```scala
val authenticatedService: Kleisli[({
    type λ[β$1$] = OptionT[Hello2.AppTask, β$1$]
  })#λ, Request[Hello2.AppTask], Response[Hello2.AppTask]]
```

Moving on, we need to fix that, so we use Router to map the empty path element "" to this service, and
add the .orNotFound to give us 404 for an unmatched string.

The final section creates the BlazeServer. The difference is that it needs an
AppEnivornment (i.e. with Authenticator) instead of the default ZIO environment. I've split this
part into 2 for clarity - first creating server1 which requires AppEnvironment, then using the
ZIO .provideSome to give it the Authenticator service

## Testing
