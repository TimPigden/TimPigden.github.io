---
layout: default
title: Simulating IoT Events - with Zio Streams
description: Illustrates use of zio streams to generate simulated events
---

# Simulating IoT Events - with Zio Streams

This is the second part of a [series](intro.md) about simulating and analysing IoT events
using Zio Streams.

In the previous blog [Speeding up Time with Zio Test](./SpeedingUpTime.md)
I looked at generating timed events using the Zio Test Clock.

In this blog I am going to make use of that to simulate IoT temperature sensors
for a refrigerated vehicle (the example is chosen because it's close to
the day job at [Optrak](https://optrak.com)

Zio streams features in this article:
- ZStream.unfoldM
- take
- run
- schedule
- Sink.collectAll
- zip
- map and mapM
- mergeAllUnbounded

I want a stream of chill events
```scala
  trait Event {
    def at: Instant
  }

  case class ChillEvent(vehicleId: String, temperature: Temperature, at: Instant) extends Event
```
To be vaguely realistic, the temperature of the vehicle at one instant is going to be
related to the temperature at the previous instant. Therefore I cannot simply
generate random events - I need to link them somehow.

In this particular case I am going to assume that temperature takes a biased "self-correcting" random
walk around the target temperature - which is the thermostat value. I'm no refrigeration
engineer but it's sufficient for testing the streaming. But I could construct alternative
models.

To implement the process I am going to use ZStream.unfoldM:
```scala
  final def unfoldM[R, E, A, S](s: S)(f0: S => ZIO[R, E, Option[(A, S)]]): ZStream[R, E, A] =
```
To create stream of type A, this takes an initial state and a function that converts the state into an Option of
the new state and A. It's an Option to allow the generation process to terminate with None.

To implement this in a general way, I create a typeclasss
```
  trait EventGenerator[Evt, S] {
    def generate(s: S): ZIO[ZEnv, Nothing, Option[(Evt, S)]]
  }
```
My events are going to be evenly spaced, so I created the convenience method:
```scala
  def generatedStream[Evt, S](initialState: S, generator: EventGenerator[Evt, S], timing: JDuration) =
    ZStream.unfoldM(initialState)(generator.generate)
    .schedule(Schedule.spaced(Duration.fromJava(timing)))
```
Note that throughout I'm using java.time with
```scala
import java.time.{Duration => JDuration}
````
as this is part of my external model. You could equally well use zio Duration or scala.concurrent.Duration.

## Example, A Biased Random Temperature Generator

My ChillEvent generator, for state, only requires the previous ChillEvent (a more complex
variant might require the previous n events). But I also want some "control" constants.
So my EventGenerator can be implemented as (though the details are not really important)
follows:
```scala
  /**
   * creates an EventGenerator implementing biased random walk. Note this is not
   * intended to emulate real refrigeration units - just give us some numbers to play with
   * @param variation maximum we should change at each tick
   * @param centre target temperature to which we are biased
   * @param bias number in range 0 - 1 representing the proportion
   *             of ticks at which we attempt to move towards the centre
   */
  def centeringRandomWalkGenerator(variation: Double, centre: Double, bias: Double): EventGenerator[ChillEvent, ChillEvent] = { s =>
    for {
      random <- getRandom
      d1a <- random.nextDouble
      d1 = d1a * 2 - 1
      d2 <- random.nextDouble
      rawAmount = if (d2 < bias) {
        // we want to move towards centre
        val direction = if (s.temperature > centre) -1 else 1
        Math.abs(d1) * direction
      } else d1
      adjustedAmount = rawAmount * variation
      nw <- now // gets the current time from, the clock
      newEvent = s.copy(temperature = adjustedAmount, at = nw)
    } yield Some((newEvent, newEvent))
  }
```

Now to actually run this to produce the stream we can use the following zio-test code:
```scala
  suite("test emitting stream")(
    testM("random walk"){
      for {
        nw <- Generators.now // utility method gets current time as instant from Clock
        initialState = ChillEvent("vehicle1", -18.0, nw)
        _ <- fastTime(JDuration.ofSeconds(60), JDuration.ofMillis(10))
        randomWalker = Generators.centeringRandomWalkGenerator(1, -10, 0.1)
        stream = EventStreams.generatedStream(initialState, randomWalker, JDuration.ofMinutes(1)).take(200)
        sink = Sink.collectAll[ChillEvent]
        runner <- stream.run(sink)
        _ <- Live.live(console.putStrLn(s"${runner.mkString("\n")}"))
      } yield {
        assert(runner.size, equalTo(200))
      }
    }
```
Inspection of the end of the console output gives:
```
ChillEvent(vehicle1,-10.849741161911801,1970-01-01T03:14:00Z)
ChillEvent(vehicle1,-11.221186839718843,1970-01-01T03:15:00Z)
ChillEvent(vehicle1,-11.066222354002646,1970-01-01T03:16:00Z)
ChillEvent(vehicle1,-11.306079659585222,1970-01-01T03:17:00Z)
ChillEvent(vehicle1,-10.626797371171312,1970-01-01T03:18:00Z)
```
So we are moving from an initial temperature of -18 to one that is centred around
-10 - our expected result.

The fastTime method is a convenience method derived from the work in the
[previous blog](./SpeedingUpTime.md)
```scala
  def fastTime(testIntervals: JDuration, liveIntervals: JDuration) =
    Live.withLive(TestClock.adjust(Duration.fromJava(testIntervals)))(
    _.repeat(Schedule.spaced(Duration.fromJava(liveIntervals)))).fork
```

# Emulating Transmission Delays

Mobile IoT sensors may take send regular readings, but they do not necessarily arrive
evenly spaced at the other end - for example my truck might be travelling though
a an area with no signal. I want to emulate that behaviour. To do this I want
a wrapper event:
```scala
  case class ReceivedEvent[Evt](event: Evt, receivedAt: Instant) extends Event {
    override def at = receivedAt
  }
```

### Simplistic Solution with mapM

One way of achieving this might be with the ZStream mapM function:
```scala
  def randomEventDelayStream[Evt <: Event](inStream: ZStream[ZEnv with Clock, Nothing, Evt]) =
    inStream.mapM { ev =>
      randomDuration(JDuration.ofMillis(10), JDuration.ofSeconds(10)).map { d =>
        val receivedTime = ev.at.plus(d)
        ReceivedEvent(ev, receivedTime)
      }
    }
```
This will randomly delay the event's receipt at the other other end.

However, this is not really what we want for a couple of reasons:
- firstly, events will now be received out of sequence - which (we pretend) is not actually the case
- secondly, there's no relationship between successive delays - whereas our model is expecting some of these to be due to out-of-signal geography.

### Generating a "delay" stream

Instead of using random delays, we could generate a stream of delays, then combine that with our original stream. Let's see how that might be done.
```scala
  case class Delay(amount: JDuration)
```
The generator itself is built using the same technique as for the ChillEvents. The logic
is not particulary important for our streams investigation but I've included for completeness.
```scala
  /**
   * arbitrarily generate delays or reduce arrays already there
   * @param howOften how often do we actually get delays
   * @param variation max delay we want to create
   * @param standardDelay all messages are delayed by a few milliseconds
   * @param sampleFrequency how frequently our source main stream set is generating events
   */
  def delayGenerator(howOften: JDuration,
                     variation: JDuration,
                     standardDelay: JDuration,
                     sampleFrequency: JDuration): EventGenerator[Delay, Delay] =
    new EventGenerator[Delay, Delay] {
      private val sampleFrequencyMillis = sampleFrequency.toMillis
      private val howOftenDbl = sampleFrequency.toMillis.toDouble / howOften.toMillis // we will use this to get probability of introducing new delay
      private val standardDelayMillis = standardDelay.toMillis

      override def generate(s: Delay): ZIO[zio.ZEnv, Nothing, Option[(Delay, Delay)]] = {
        val sMillis = s.amount.toMillis
        val newMillis = if (sMillis > sampleFrequency.toMillis)
          IO.succeed(JDuration.ofMillis(sMillis - sampleFrequency.toMillis))
        else if (sMillis > standardDelayMillis)
          IO.succeed(standardDelay)
        else for {
          r <- randomDouble
          newDelay <- if (r > howOftenDbl) // we don't want a new one
            IO.succeed(standardDelay)
          else randomDuration(standardDelay, variation)
        } yield newDelay
        newMillis.map { nd =>
          Some((Delay(nd), Delay(nd)))
        }
      }
    }

```
We can test it with:
```scala
    testM("run delay"){
      val initialState = Delay(JDuration.ofMillis(0))
      val delayer = Generators.delayGenerator(howOften = JDuration.ofMinutes(10),
        variation = JDuration.ofSeconds(300),
        standardDelay = JDuration.ofMillis(10),
        sampleFrequency = JDuration.ofSeconds(20)
      )
      val delays = ZStream.unfoldM(initialState)(delayer.generate).take(200)
      val sink = Sink.collectAll[Delay]
      for {
        runner <- delays.run(sink)
        _ <- Live.live(console.putStrLn(s"${runner.mkString("\n")}"))
      } yield {
        assert(runner.size, equalTo(200))
      }
    },
```
Inspection of some of the output shows that most delays are 0.01s but at one
point we hit the sequence:
```
Delay(PT0.01S)
Delay(PT3M40.105S)
Delay(PT3M20.105S)
Delay(PT3M0.105S)
Delay(PT2M40.105S)
Delay(PT2M20.105S)
Delay(PT2M0.105S)
Delay(PT1M40.105S)
Delay(PT1M20.105S)
Delay(PT1M0.105S)
Delay(PT40.105S)
Delay(PT20.105S)
Delay(PT0.105S)
Delay(PT0.01S)
```
which shows the output we want - successive delays of 20s less than the previous delay. For
a reporting cycle of 20s this will effectively lead to a bunch of consecutive signals
arriving immediately one after the other - as our truck comes back into signal range.

### Combining the results

```scala
    testM("run delayed chills"){
      val initialDelay = Delay(JDuration.ofMillis(0))
      val delayer = Generators.delayGenerator(howOften = JDuration.ofMinutes(10),
        variation = JDuration.ofSeconds(300),
        standardDelay = JDuration.ofMillis(10),
        sampleFrequency = JDuration.ofSeconds(20)
      )
      val delays = ZStream.unfoldM(initialDelay)(delayer.generate)
      for {
        _ <- fastTime(JDuration.ofSeconds(10), JDuration.ofMillis(5))
        initialChill <- Support.initialiseState
        randomWalker = Generators.centeringRandomWalkGenerator(1, -10, 0.1)
        chillStream = EventStreams.generatedStream(initialChill, randomWalker, JDuration.ofSeconds(20))
        receivedEvents = delays.zip(chillStream).map { pair =>
          ReceivedEvent(pair._2, pair._2.at.plus(pair._1.amount))
        }.take(200)
        sink = Sink.collectAll[ReceivedEvent[ChillEvent]]
        runner <- receivedEvents.run(sink)
        _ <- Live.live(console.putStrLn(s"${runner.mkString("\n")}"))
      } yield {
        assert(runner.size, equalTo(200))
      }
    }
```
Having created the two streams - "delays" and "chillStream", we zip them together to
create stream of tuples (Delay, ChillEvent), then map that to a ReceivedEvent.

Note that we can apply the take(200) to the zipped and mapped stream. Zio Streams is pull-based
so we will only create 200 of each input stream.

The results are as expected - here is a snippet that shows the injected delay, with the
messages all bunching up as they are received almost at the same time before the situation reverts
to normal.

```
ReceivedEvent(ChillEvent(vehicle1,-9.905406499455315,1970-01-01T00:55:40Z),1970-01-01T00:55:40.010Z)
ReceivedEvent(ChillEvent(vehicle1,-10.647473478986841,1970-01-01T00:56:00Z),1970-01-01T00:56:00.010Z)
ReceivedEvent(ChillEvent(vehicle1,-10.698151314483637,1970-01-01T00:56:20Z),1970-01-01T01:00:01.813Z)
ReceivedEvent(ChillEvent(vehicle1,-10.686673916066201,1970-01-01T00:56:40Z),1970-01-01T01:00:01.813Z)
ReceivedEvent(ChillEvent(vehicle1,-10.918241191699977,1970-01-01T00:57:00Z),1970-01-01T01:00:01.813Z)
ReceivedEvent(ChillEvent(vehicle1,-10.481735903440713,1970-01-01T00:57:20Z),1970-01-01T01:00:01.813Z)
ReceivedEvent(ChillEvent(vehicle1,-9.908628147897678,1970-01-01T00:57:40Z),1970-01-01T01:00:01.813Z)
ReceivedEvent(ChillEvent(vehicle1,-9.408665619828056,1970-01-01T00:58:00Z),1970-01-01T01:00:01.813Z)
ReceivedEvent(ChillEvent(vehicle1,-9.863295302276958,1970-01-01T00:58:20Z),1970-01-01T01:00:01.813Z)
ReceivedEvent(ChillEvent(vehicle1,-8.908266245642997,1970-01-01T00:58:40Z),1970-01-01T01:00:01.813Z)
ReceivedEvent(ChillEvent(vehicle1,-8.892048354963988,1970-01-01T00:59:00Z),1970-01-01T01:00:01.813Z)
ReceivedEvent(ChillEvent(vehicle1,-7.9458962709539795,1970-01-01T00:59:20Z),1970-01-01T01:00:01.813Z)
ReceivedEvent(ChillEvent(vehicle1,-8.094473115266439,1970-01-01T00:59:40Z),1970-01-01T01:00:01.813Z)
ReceivedEvent(ChillEvent(vehicle1,-8.55209550309182,1970-01-01T01:00:00Z),1970-01-01T01:00:01.813Z)
ReceivedEvent(ChillEvent(vehicle1,-9.366095284824228,1970-01-01T01:00:20Z),1970-01-01T01:00:20.010Z)
```

## Simulating Multiple Vehicles

Finally, we want to emulate the results of an IoT "system" returning results from a small fleet of vehicles.
We would expect the system to be sending all results back in a unified stream (in the next blog,
we'll use Kafka to send the data from the IoT collection to the analytic backend).

Conceptually, this is straight-forward
* create a stream per vehicle
* run them in parallel
* combine the results.

We expect the results to be more-or-less time-ordered on "receivedAt" but so long as they are
ordered within each vehicle this is not critical as there is no expected causal relationship
between refrigeration units on different vehicles.

The following code is largely a rearrangement of the previous examples:
```scala
    testM("multiple vehicles"){
      val initialDelay = Delay(JDuration.ofMillis(0))
      val delayer = Generators.delayGenerator(howOften = JDuration.ofMinutes(10),
        variation = JDuration.ofSeconds(300),
        standardDelay = JDuration.ofMillis(10),
        sampleFrequency = JDuration.ofSeconds(20)
      )
      val randomWalker = Generators.centeringRandomWalkGenerator(1, -10, 0.1)

      def vehicleStream(i: Int, startAt: Instant) = {
        val initialChill = ChillEvent(s"v-$i", -18.0, startAt)
        val chillStream = EventStreams.generatedStream(initialChill, randomWalker, JDuration.ofSeconds(20))
        val delays = ZStream.unfoldM(initialDelay)(delayer.generate)
        delays.zip(chillStream).map { pair =>
            ReceivedEvent(pair._2, pair._2.at.plus(pair._1.amount))
          }
        }

      for {
        _ <- fastTime(JDuration.ofSeconds(10), JDuration.ofMillis(5))
        nw <- Generators.now
        streams = 1.to(20).map { v => vehicleStream(v, nw)}
        combined = ZStream.mergeAllUnbounded()(streams:_*)
        sink = Sink.collectAll[ReceivedEvent[ChillEvent]]
        runner <- combined.take(2000).run(sink)
        _ <- Live.live(console.putStrLn(s"${runner.mkString("\n")}"))
      } yield {
        assert(runner.size, equalTo(2000))
      }
    }
  )
```
The only new feature is that we are now creating multiple streams and using
ZStream.mergeAllUnbounded to collect them into a single stream.

Sample output:
```
ReceivedEvent(ChillEvent(v-2,-14.626272506447977,1970-01-01T00:33:00Z),1970-01-01T00:33:00.010Z)
ReceivedEvent(ChillEvent(v-18,-11.758463344137432,1970-01-01T00:33:00Z),1970-01-01T00:33:00.010Z)
ReceivedEvent(ChillEvent(v-8,-9.59459141052508,1970-01-01T00:33:00Z),1970-01-01T00:34:34.810Z)
ReceivedEvent(ChillEvent(v-13,-17.175593237140184,1970-01-01T00:33:00Z),1970-01-01T00:33:00.010Z)
```

# Source code
Source code can be found on [github](https://github.com/TimPigden/zio-http4s-examples) in the
streams subproject





