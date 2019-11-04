---
layout: default
title: Generating Chill Events - with State and unfoldM
description: Illustrates use of unfoldM to generate related events
---

# Generating Chill Events
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
  def generatedStream[Evt, S](initialState: S, generator: EventGenerator[Evt, S], every: Duration) =
    ZStream.unfoldM(initialState)(generator.generate)
      .schedule(Schedule.spaced(every))
```

## Example, A Biased Random Temperature Generator

My ChillEvent generator, for state, only requires the previous ChillEvent (a more complex
variant might require the previous n events). But I also want some "control" constants.
So my EventGenerator can be implemented as
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








