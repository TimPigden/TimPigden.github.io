---
layout: default
title: ZIO and Http4s with auth, codecs and zio-test: Part2
description: second part of zio http4s blog
---

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

## Authenticator

An Authenticator is Module pattern to provide an Authenticator.Service. The service takes a userName and password and returns Task[AuthToken]

```scala
object Authenticator {

  type Authenticator = Has[Service]

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

  val friendly = ZLayer.succeed(friendlyAuthenticator)
}

package object authenticator {
  def authenticate(userName: String, password: String): RIO[Authenticator, AuthToken]
  = ZIO.accessM[Authenticator](_.get.authenticate(userName, password))
}
```

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
    userNamePasswordOpt.map { asSplit =>
      val tok = authenticator.authenticate(asSplit(0), asSplit(1))
      tok.either
    }.getOrElse(unauthenticated)
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
  type AppEnvironment = Authenticator with Clock

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

  val server = server1.provideCustomLayer(friendly)

  def run(args: List[String]): ZIO[ZEnv, Nothing, Int] =
    server.foldM(err => putStrLn(s"execution failed with $err") *> ZIO.succeed(1), _ => ZIO.succeed(0))
}
```

This has suddenly got rather more complicated.

Our App is extended with AuthenticationMiddleware, which means our types line up.

We create the new hello2Service instance of the right type parameterisation.

Next we wrap our hello2Service in our authenticationMiddleware. The result of this operation is not an HttpRoutes - in fact intellij says it's
```scala
val authenticatedService: Kleisli[({
    type λ[β$1$] = OptionT[Hello2.AppTask, β$1$]
  })#λ, Request[Hello2.AppTask], Response[Hello2.AppTask]]
```

Moving on, we need to fix that, so we use Router to map the empty path element "" to this service, and
add the .orNotFound to give us 404 for an unmatched string.

The final section creates the BlazeServer. And it needs our authentication environment - so we provide the custom layer **friendly** from our Authenticator

## Testing

For testing, we will just test against *hello2Service*. "e need a mechanism to insert authentication into the requests.
Back in AuthenticationHeaders, I created just such a method:
```scala
object AuthenticationHeaders {
  def addAuthentication[Tsk[_]](request: Request[Tsk], username: String, password: String): Request[Tsk] =
    request.withHeaders(request.headers.put(Header("Authorization", s"$username $password")))
}
```
Unlike Hello1Service, which was a static object, Hello2Service was created as a class since it
is likely to be used with different *R* environment values. For testing, I've created an object
to provide the service to test.
```scala
object Middlewares {
  val withMiddleware = new AuthenticationMiddleware {
    override type AppEnvironment = Authenticator
  }

  val hello2Service1 = new Hello2Service[Authenticator]
  
  val hello2Service = Router[withMiddleware.AppTask](
  ("" -> withMiddleware.authenticationMiddleware(hello2Service1.service)))
    .orNotFound
}
```
Essentially this mirrors the code in our Hello2 server.

This is then called directly from our individual tests:
```scala
 suite("routes suite")(

    testM("root request returns forbidden") {
      val io = hello2Service.run(Request[withMiddleware.AppTask](Method.GET, uri"/"))
      assertM(io.map(_.status))(
        equalTo(Status.Forbidden)) // will fail if nothing there
    },

    testM("root request with authentication returns ok") {
      val req1 = Request[withMiddleware.AppTask](Method.GET, uri"/")
      val req = AuthenticationHeaders.addAuthentication(req1, "tim", "friend")
      val io = hello2Service.run(req).provideCustomLayer(Authenticator.friendly)
      assertM(io.map(_.status))(equalTo(Status.Ok)) // will fail if nothing there
    }
    ,
    testM("unmapped request returns not found") {
      val req1 = Request[withMiddleware.AppTask](Method.GET, uri"/a")
      val req = AuthenticationHeaders.addAuthentication(req1, "tim", "friend")
      val io = hello2Service.run(req)
      assertM(io.map(_.status))(equalTo(Status.NotFound))
    }
    ,
    testM("root request body returns hello!") {
      val req1 = Request[withMiddleware.AppTask](Method.GET, uri"/")
      val req = AuthenticationHeaders.addAuthentication(req1, "tim", "friend")
      val io = hello2Service.run(req)
      val iop = (for {
        request <- io
        body <- request.body.compile.toVector.map(x => x.map(_.toChar).mkString(""))
      } yield body)
      assertM(iop)(equalTo("hello! tim"))
    }
    ,
    testM("bad password gives forbidden") {
      val req1 = Request[withMiddleware.AppTask](Method.GET, uri"/")
      val req = AuthenticationHeaders.addAuthentication(req1, "tim", "frond")
      val io = hello2Service.run(req).provideCustomLayer(Authenticator.friendly)
      assertM(io.map(_.status))(equalTo(Status.Forbidden))
    }

  ).provideCustomLayerShared(Authenticator.friendly)
```
Note, all tests are collectively supplied with the same Authenticator layer.
