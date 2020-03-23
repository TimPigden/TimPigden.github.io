---
layout: default
title: Example of ZLayers being used in combination
description: examp[le tests]
---

# ZIO ZLayers In Combination

The new ZLayer feature of ZIO 1.0.0-RC18+ is a great improvement on the earlier module pattern, making the addition of new services much quicker and easier. However, when used in practice I found it took a little while to get the hang of the idiom.

So below is an annotated sample of the final version of my test code which explores a number of combinations. Many thanks to Adam Fraser for help in getting this nice and streamlined.

The services are deliberate pretty simple, so hopefully it will be clear enough for a quick read.

I'm assuming you've a basic idea of ZIO test and also have read the main page on modules at

The code is all run from zio tests and is a single file. 

 Here's the top bit:
```scala
import zio._
import zio.test._
import zio.random.Random
import Assertion._

object LayerTests extends DefaultRunnableSpec {

  type Names = Has[Names.Service]
  type Teams = Has[Teams.Service]
  type History = Has[History.Service]

  val firstNames = Vector( "Ed", "Jane", "Joe", "Linda", "Sue", "Tim", "Tom")

```

## Names
So now we get to our first Service - Names
```scala
  type Names = Has[Names.Service]

  object Names {
    trait Service {
      def randomName: UIO[String]
    }

    case class NamesImpl(random: Random.Service) extends Names.Service {
      println(s"created namesImpl")
      def randomName = 
        random.nextInt(firstNames.size).map(firstNames(_))
    }

    val live: ZLayer[Random, Nothing, Names] =
      ZLayer.fromService(NamesImpl)
  }

  package object names {
    def randomName = ZIO.accessM[Names](_.get.randomName)
  }
```

This follows the typical module pattern.
* declare Names as a type alias to the Has
* in the object define Service as a trait
* create an implementation (of course you can have several)
* create a ZLayer within the object for the given implementation. ZIO convention tends to call these live
* Adds a package object which provides a useful shortening for the access

The **live** uses ZLayer.fromService - which is defined as:
```scala
  def fromService[A: Tagged, B: Tagged](f: A => B): ZLayer[Has[A], Nothing, Has[B]
```
Ignoring the Tagged (it's required for the whole Has/Layers stuff to work) you can see it takes a function f: A => B - which in this case is simply the case class constructor for NamesImpl.

As you can see Names requires the zio environmental Random to function. 

Here's a test:
```scala
def namesTest = testM("names test") {
    for {
      name <- names.randomName
    }  yield {
      assert(firstNames.contains(name))(equalTo(true))
    }
  }
```
This uses the ZIO.accessM to pull the Names from the environment. the _.get extracts the Service.

To make this work we provide the Names to the test like so:
```scala
    suite("needs Names")(
       namesTest
    ).provideCustomLayer(Names.live),
```

provideCustomLayer adds the Names layer to the existing environment

## Teams
The point about Teams is to test dependencies between modules that we've created. 
```scala
  object Teams {
    trait Service {
      def pickTeam(size: Int): UIO[Set[String]]
    }

    case class TeamsImpl(names: Names.Service) extends Service {
      def pickTeam(size: Int) = 
        ZIO.collectAll(0.until(size).map { _ => names.randomName}).map(_.toSet ) // yeah I know team could have < size!      
    }

    val live: ZLayer[Names, Nothing, Teams] =
      ZLayer.fromService(TeamsImpl)

  }
```
Teams will pick a team from the available names, making _size_ selections. 

Following Module usage patterns, although **pickTeam** needs a Names to function, we don't put it in as a ZIO[Names, Nothing, Set[String]] - instead we hold a reference in the **TeamsImpl**

Our first test is straight-forward
```scala
  def justTeamsTest = testM("small team test") {
    for {
      team <- teams.pickTeam(1)
    }  yield {
      assert(team.size)(equalTo(1))
    }
  }
```

To run this we need to give it a Teams layer:
```scala
    suite("needs just Team")(
      justTeamsTest
    ).provideCustomLayer(Names.live >>> Teams.live),
```
So what's the ">>>"?

This is the vertical composition. It show that we need a Names layer which needs a Teams layer.

However, running this, there's a slight problem.
```
created namesImpl
created namesImpl
[32m+[0m individually
  [32m+[0m needs just Team
    [32m+[0m small team test
[36mRan 1 test in 225 ms: 1 succeeded, 0 ignored, 0 failed[0m
```
Looking back to the definition of NamesImpl
```
    case class NamesImpl(random: Random.Service) extends Names.Service {
      println(s"created namesImpl")
      def randomName = 
        random.nextInt(firstNames.size).map(firstNames(_))
    }
```
So our NamesImpl is being created twice. What does that mean if our service is holding some application-unique system resource? Well actually, it turns out that the problem isn't with the Layers mechanism at all - the layers are memoized and not created multiple times in the dependency graph. It's actually an artifact of the test environment.

Changing our test suite to:
```scala
    suite("needs just Team")(
      justTeamsTest
    ).provideCustomLayerShared(Names.live >>> Teams.live),
```
fixes the problem - meaning the layer only gets created once within the test

The **justTeamsTest** requires just teams. But what if I wanted access to Teams and Names?
```scala
  def inMyTeam = testM("combines names and teams") {
    for {
      name <- names.randomName
      team <- teams.pickTeam(5)
      _ = if (team.contains(name)) println("one of mine")
        else println("not mine")
    } yield assertCompletes
  }
  ```
  To make this work we need to provide both - which is achieved by the following:
  ```scala
       suite("needs Names and Teams")(
       inMyTeam
    ).provideCustomLayer(Names.live ++ (Names.live >>> Teams.live)),
```
Here we are using the **++** combinator to create a Names with Teams layer. Note the operator precedence and extra brackets around 
```scala
(Names.live >>> Teams.live)
```
It caught me out first time round, since the compiler will otherwise not do the right thing.

## History
History is just a bit more complicated.
```scala
    object History {
    
    trait Service {
      def wonLastYear(team: Set[String]): Boolean
    }

    case class HistoryImpl(lastYearsWinners: Set[String]) extends Service {
      def wonLastYear(team: Set[String]) = lastYearsWinners == team
    }
    
    val live: ZLayer[Teams, Nothing, History] = ZLayer.fromServiceM { teams => 
      teams.pickTeam(5).map(nt => HistoryImpl(nt))
    }
    
  }
```
The constructor HistoryImpl requires a Set of names. But the only way to get one of those is to extract it from Teams. And that requires a ZIO - so we use ZLayer.fromServiceM to give us what we need.

The test follows the same pattern as before:
```scala
  def wonLastYear = testM("won last year") {
    for {
      team <- teams.pickTeams(5)
      ly <- history.wonLastYear(team)
    } yield assertCompletes
  }

    suite("needs History and Teams")(
      wonLastYear
    ).provideCustomLayerShared((Names.live >>> Teams.live) ++ (Names.live >>> Teams.live >>> History.live))
```  

And that's it.

Full source code below:
```scala
import zio._
import zio.test._
import zio.random.Random
import Assertion._

object LayerTests extends DefaultRunnableSpec {

  type Names = Has[Names.Service]
  type Teams = Has[Teams.Service]
  type History = Has[History.Service]

  val firstNames = Vector( "Ed", "Jane", "Joe", "Linda", "Sue", "Tim", "Tom")

  object Names {
    trait Service {
      def randomName: UIO[String]
    }

    case class NamesImpl(random: Random.Service) extends Names.Service {
      println(s"created namesImpl")
      def randomName = 
        random.nextInt(firstNames.size).map(firstNames(_))
    }

    val live: ZLayer[Random, Nothing, Names] =
      ZLayer.fromService(NamesImpl)
  }
  
  object Teams {
    trait Service {
      def pickTeam(size: Int): UIO[Set[String]]
    }

    case class TeamsImpl(names: Names.Service) extends Service {
      def pickTeam(size: Int) = 
        ZIO.collectAll(0.until(size).map { _ => names.randomName}).map(_.toSet ) // yeah I know team could have < size!      
    }

    val live: ZLayer[Names, Nothing, Teams] =
      ZLayer.fromService(TeamsImpl)

  }
  
  object History {
    
    trait Service {
      def wonLastYear(team: Set[String]): Boolean
    }

    case class HistoryImpl(lastYearsWinners: Set[String]) extends Service {
      def wonLastYear(team: Set[String]) = lastYearsWinners == team
    }
    
    val live: ZLayer[Teams, Nothing, History] = ZLayer.fromServiceM { teams => 
      teams.pickTeam(5).map(nt => HistoryImpl(nt))
    }
    
  }
  

  def namesTest = testM("names test") {
    for {
      name <- names.randomName
    }  yield {
      assert(firstNames.contains(name))(equalTo(true))
    }
  }

  def justTeamsTest = testM("small team test") {
    for {
      team <- teams.pickTeam(1)
    }  yield {
      assert(team.size)(equalTo(1))
    }
  }
  
  def inMyTeam = testM("combines names and teams") {
    for {
      name <- names.randomName
      team <- teams.pickTeam(5)
      _ = if (team.contains(name)) println("one of mine")
        else println("not mine")
    } yield assertCompletes
  }
  
  
  def wonLastYear = testM("won last year") {
    for {
      team <- teams.pickTeam(5)
      _ <- history.wonLastYear(team)
    } yield assertCompletes
  }

  val individually = suite("individually")(
    suite("needs Names")(
       namesTest
    ).provideCustomLayer(Names.live),
    suite("needs just Team")(
      justTeamsTest
    ).provideCustomLayer(Names.live >>> Teams.live),
     suite("needs Names and Teams")(
       inMyTeam
    ).provideCustomLayer(Names.live ++ (Names.live >>> Teams.live)),
    suite("needs History and Teams")(
      wonLastYear
    ).provideCustomLayerShared((Names.live >>> Teams.live) ++ (Names.live >>> Teams.live >>> History.live))
  )
  
  val altogether = suite("all together")(
      suite("needs Names")(
       namesTest
    ),
    suite("needs just Team")(
      justTeamsTest
    ),
     suite("needs Names and Teams")(
       inMyTeam
    ),
    suite("needs History and Teams")(
      wonLastYear
    ),
  ).provideCustomLayerShared(Names.live ++ (Names.live >>> Teams.live) ++ (Names.live >>> Teams.live >>> History.live))

  override def spec = (
    individually
  )
}

import LayerTests._

package object names {
  def randomName = ZIO.accessM[Names](_.get.randomName)
}

package object teams {
  def pickTeam(nPicks: Int) = ZIO.accessM[Teams](_.get.pickTeam(nPicks))
}
  
package object history {
  def wonLastYear(team: Set[String]) = ZIO.access[History](_.get.wonLastYear(team))
}

```
If you have any more complex requirements, ask on Discord in #zio-users or check out the main zio web page and [docs](https://zio.dev)








