---
layout: default
title: Streaming example using IoT Sensor Emulation
description: A set of blogs that explore ZIO Streams and other features to build
and process results of an IoT emulator for a truck-based sensors
---

# ZIO Streams using IoT Sensor Emulation

This is a series of blog posts using ZIO to create a simulator for IoT sensors.

The basic idea is that we will generate streams of data emulating the behaviour
of multiple sensors, collect them together, store in Kafka, retrieve and finally
carry out some simple streaming analytics.

My example will be Frozen Food semi-trailer temperature sensors. We're going to emulate various
temperature variations, GPS signal generation and so on.

The [first article](./SpeedingUpTime.md) will look at generating the IoT event times and because we
want to generate them over a day and don't want to wait all day to run the simulation
we must find a way to run in an "alternate reality" where time passes faster.

