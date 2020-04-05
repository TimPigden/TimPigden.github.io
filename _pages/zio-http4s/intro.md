---
layout: default
title: ZIO, Http4s, Auth, Codecs and zio-test
description: Examples of use of ZIO with the http4s library, illustrating http4s authentication, custom codes and testing with zio-test
---
# Updated for ZIO 1.0.0-RC18-2 and HTTP4s 0.21.3

This has been updated for latest zio and http4s. There is also an intermediate zio rC18-2 + http4s 0.20.21 in the github on an obviously named branch.

The document is an update which I've pushed out in a slightly hurried manner. If there are any errors relating to residual stuff from previous releases please drop me a line and I'll fix. The source code compiles and runs.

### Why Another ZIO/Http4s Client Example?

There are a couple of sample projects already in existence showing you how to use zio with http4s as a server. So why another?

What this adds to the mix are 3 things that might be independently useful:
* an http4s Authentication/Authorization example
* an example of custom encoding and decoding.
* Tests written using the zio-test module.

For a detailed description of the various libraries you should head on over to the relevant pages. However, if, like me, you sometimes struggle a bit to get everything working together, then this may help.
Also, I find some of the other zio coding examples a little daunting, as authors cram multiple concepts into a small sample.
These examples are rather more spread out. If this means it's a little dull for some of you, please go straight to the source code on github!

I don't claim to be an expert in this stuff. But we do have 4 microservices using this combination already in production and
although processing volumes are low, they do seem to work!

I will not attempt to explain either of these libraries, but briefly, ZIO is the latest in a series of scala effects libraries which includes cats.IO and Monix.
Http4s is a popular typelevel web framework based on the Blaze server (there is also a client).

### These examples

The [github project](https://github.com/TimPigden/zio-http4s-examples) contains 4 sets of services that can be treated as a progression from simplest to most complex:

* Hello1 is a very basis "hello world" web service
* Hello2 adds an authentication layer.
* Hello3 is expands 1 with a custom encoding and decoding of an xml object, using scala xml
* Hello4 combines 2 & 3

The blog is broken into
* [ZIO, http4s Part 1 - basic Http4s Server and Service testing](zio-http4s-part1.md).
* [ZIO, http4s Part 2 - Testing with http4s Client](zio-http4s-part2.md).
* [ZIO, http4s Part 3 - Adding Authentication](zio-http4s-part3.md).
* [ZIO, http4s Part 4 - Custom Codecs](zio-http4s-part4.md).
