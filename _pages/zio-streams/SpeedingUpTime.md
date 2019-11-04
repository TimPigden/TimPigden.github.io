---
layout: default
title: Speeding up time with Zio TestClock
description: Illustrates use of TestClock and a bit of streaming
---

# Speeding up Time

For my IoT sensor emulator, one of the most crucial elements is the timestamp.

A syntheised stream of events needs the events to have a timestamp and we can
generate that using ZIO's clock.

But first my event type:
```scala
  case class SimpleEvent(at: Instant)
```
I've chosen to use the java.time.Instant as its nice and straight-forward. Our sensors are
going to be mobile and may cross timezones. We don't need the time to be a proxy for location
(and guessing location of timezone is never going to be terribly useful).

We will ignore the other data that will be needed for the Event for the time-being (these will
come later!).

## Creating the Stream

So how do we generate a stream of SimpleEvent?

Firstly, though, where do we get the Instant from?

Well we are simulating data, so using the real clock is not going to be very good - we'd get
different results every time we used the system. And of course, if we want a full day's worth
of data it will take a full day.

Now we could just generate a bunch of Instants in a collection and initialize the stream from them
using
```scala
  ZStream.fromIterable(myInstantCollection)
```

This would work, but it would require the entire set of instants to be defined in advance, and
since this blog is about streams and clocks we'll do it another way!

ZIO includes a Clock in its standard environment and a TestClock in the standard
test environment. And we can use the latter to generate our timestamp, by fast-forwarding
it.

This is our generator:
```scala
  def myStream = ZStream.repeatEffect(
      ZIO.accessM[Clock](_.clock.currentDateTime)
      .map(at => SimpleEvent(at.toInstant))
    )
    .schedule(Schedule.spaced(Duration.fromScala(10.seconds)))
```
So this illustrates a few things:

Working from inside out:
```scala
ZIO.accessM[Clock](_.clock.currentDateTime)
```
 is used to extract the clock from the current environment (at this stage it
could be a regular clock or a TestClock - we don't care) and use clock's currentDateTime to get
the OffsetDateTime. OffsetDateTime provides the "instant" value plus the zone
for which it applies. We don't care about timeZones so we just convert it to an instant.

Now ZStream.repeatEffect takes an effect and runs it forever - as fast as the
system will let it.
We don't need that so we can use
```scala
    .schedule(Schedule.spaced(Duration.fromScala(10.seconds)))
```
to ensure we only get one every 10 seconds.

### Running it - first attempt

Below is our first attempt at running the code
```scala
object SpeedingUpTime extends DefaultRunnableSpec(
  suite("timings")(
    testM("first attempt"){
      val stream = myStream.take(30)
      val sink = Sink.collectAll[SimpleEvent]
      for {
        runner <- stream.run(sink)
      } yield assert(runner.size, equalTo(30))
    }
  )
)
```
So I'm assuming you've already looked at ZIO test and know what DefaultRunnableSpec, suite
and testM all do.

So the important points are:
```scala
      val stream = myStream.take(30)
```
We create the stream but tell the system we only want to take the first 30 elements.

```scala
      val sink = Sink.collectAll[SimpleEvent]
```
We create a sink for SimpleEvents

```scala
      for {
        runner <- stream.run(sink)
      } yield assert(runner.size, equalTo(30))
```
We actually run the stream and check it has 30 events.

So at this point, given the above, you might expect a stream to be run and the
test to return true. We'd expect the process to take 300 seconds more or less,
because 30 events, one every 10 seconds.

But that's not what happens. Instead the system just hangs (and eventually Zio test asks
if you really expected it to last longer than a minute).

## Test vs Live

So what's going on?

The issue is that the philosophy of zio-test is that all the system components are mocked objects and don't necessarily do what you expect.
The following snippet is from zio.test.environment:
```scala
case class TestEnvironment(
  blocking: Blocking.Service[Any],
  clock: TestClock.Test,
  console: TestConsole.Test,
  live: Live.Service[ZEnv],
  random: TestRandom.Test,
  scheduler: TestClock.Test,
  sized: Sized.Service[Any],
  system: TestSystem.Test
) extends Blocking
    with Live[ZEnv]
    with TestClock
    with TestConsole
    with TestRandom
    with TestSystem
    with Scheduler
    with Sized
```
So we've got a TestClock, a TestConsole, TestRandom and TestSystem.

Our focus here is going to be on TestClock, but a quick note on the others:

#### TestRandom
This allows you to modify the behaviour of the Random service, inject your own numbers and so on. See the API docs/source code comments
for more details. It defaults to normal Random behaviour if you do
nothing special with it.

#### TestConsole
This is for testing writing to and reading from the console. Stuff written to it doesn't actually get printed out on the console - instead
there are input and output buffer that you can inspect and manipulate.

#### TestSystem
To quote from the api docs:

*TestSystem supports deterministic testing of effects involving system  properties. Internally, `TestSystem` maintains mappings of environment
variables and system properties that can be set and accessed. No actual environment variables or
system properties will be accessed or set as a result of these actions.*

There is no TestBlocking.

### Live
Lurking in the background there is the Live environment. Not everything is a test and not everything wants to be mocked.
Your test program has access to both Test and Live.

For example, within the test program  we can print messages to the real console by accessing Live. This prints
out my event:
```scala
      _ <- Live.live(console.putStrLn(s"at $evt"))
```

### TestClock

With TestClock, your program is in charge of the advance of time. Left to itself, TestClock does nothing to advance events
and is initialised to 00:00 1/1/70.

We can see the impact of this by removing the schedule from our stream and printing the events as they are created to the
live console:
```scala
  def myStream = ZStream.repeatEffect(
    for {
      at <- ZIO.accessM[Clock](_.clock.currentDateTime)
      evt = SimpleEvent(at.toInstant)
      _ <- Live.live(console.putStrLn(s"at $evt"))
    } yield evt
  )
//    .schedule(Schedule.spaced(Duration.fromScala(10.seconds)))
```
The program now runs with output of (30 lines):
```
at SimpleEvent(1970-01-01T00:00:00Z)
at SimpleEvent(1970-01-01T00:00:00Z)
```

What we want to do is advance the test clock so that it thinks it has hit 10 seconds from the start to trigger our schedule and
repeat for as long as we need.

But advancing the clock cannot take place on the same fiber (zio has fibers instead of threads, see the docs). So instead we fork
a new fiber to give us the fast-forward:
```scala
    testM("sepaate ticker"){
      val stream = myStream.take(30)
      val sink = Sink.collectAll[SimpleEvent]
      for {
        _ <- TestClock.adjust(Duration.fromScala(1.seconds))
        .repeat(Schedule.recurs(300)).fork
        runner <- stream.run(sink)
      } yield assert(runner.size, equalTo(30))
    }
  )
```
This gives
```
at SimpleEvent(1970-01-01T00:05:01Z)
at SimpleEvent(1970-01-01T00:05:01Z)
at SimpleEvent(1970-01-01T00:05:01Z)
```
Not quite what we wanted - our fast forward is too fast and by the time the stream generator grabs the test clock current
time, it's moved all the way to the "end" of our 300 seconds.

So we need to slow down the rate at which our test clock moves forward. Instead of using Schedule.recurs we can use Schedule.spaced

But there's a problem here, Schedule.spaced needs a clock to do the spacing. But it can't use the test clock, since that's the thing we're
trying to change. Instead we want to use the Live clock. To do this we use the withLive function
```scala
        _ <- Live.withLive(TestClock.adjust(Duration.fromScala(1.seconds)))(
          _.repeat(Schedule.spaced(Duration.fromScala(10.millis)))).fork
```

In the Live source this is defined as
```scala
  /**
   * Provides a transformation function with access to the live environment
   * while ensuring that the effect itself is provided with the test
   * environment.
   */
  def withLive[R, R1, E, E1, A, B](zio: ZIO[R, E, A])(f: IO[E, A] => ZIO[R1, E1, B]): ZIO[R with Live[R1], E1, B]
```
So our first argument is the effect (adjusting the clock) and the second uses Live environment to schedule the repeat
of that effect.

```
at SimpleEvent(1970-01-01T00:00:18Z)
at SimpleEvent(1970-01-01T00:00:32Z)
at SimpleEvent(1970-01-01T00:00:32Z)
at SimpleEvent(1970-01-01T00:00:32Z)
at SimpleEvent(1970-01-01T00:00:40Z)
at SimpleEvent(1970-01-01T00:00:50Z)
at SimpleEvent(1970-01-01T00:01:00Z)
at SimpleEvent(1970-01-01T00:01:10Z)
at SimpleEvent(1970-01-01T00:01:20Z)
```
After the first 4 "ticks" (40 mills real time) where, presumably, things are initialising, the system settles to a steady state of 1 second
test time to 10 mills of real time.

## Use Beyond the Test
Using this technique, we actually have a fixed ratio of "real" to "test" time - in this case 100:1.

While TestClock is automatically part of the test environment, it is not bound to DefaultRunnableSpec. Consequently, what we have here is a
general mechanism that could be applied to any simulation of real-time events that would benefit from a controllable passage of time.

## Finally
I'd like to thank Adam Fraser especially for help with this testing.

Full source code can be found at on [github](https://github.com/TimPigden/zio-http4s-examples/blob/master/streams/src/test/scala/tsp/stream/events/SpeedingUpTime.scala)







