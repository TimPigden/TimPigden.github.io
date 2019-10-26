---
layout: default
title: Magnolia typeclass generation for Avro
description: Examples of use of magnolia library to generate schema, reader and writer typeclasses for Avro serialization
---

# Magnolia for Avro

This describes a library I've written for serialization of scala data into
Avro format.

[Magnolia](https://propensive.com/opensource/magnolia/tutorial) is a macro-based automatic
typeclass generator, used in a number of projects. It ultimately provides similar functionality
to the [shapeless](https://github.com/milessabin/shapeless) typeclass generation.
It can be used to automatically generate typeclasses for case classes and sealed traits. Magnolia has
a good tutorial so I won't attempt to describe how it works.

[Avro](https://avro.apache.org/) is an apache project for data serialization. It is normally used as a binary
format serializer (json is an option) and is often associated with [Kafka](https://kafka.apache.org/)
streaming platform, which is how it will be used later in this blog series.

Avro binary format is schema based. Unlike json or xml, in which fields are tagged with the field
name in each record, in Avro the binary format is pure data. The schema is required to construct and
extract data from the binary records.

Avro, and the tooling around it (e.g. from [Confluent](https://www.confluent.io/)),
provides mechanisms for schema migration and schema registry. Schema migration allows
users of one version of a schema to read data produced by another
version, by filling in defaults or transforming values. However, we will not be looking
at that, here.

Other tooling allows automatic generation of reading and writing classes - in java for the jvm, but also
for other languages. There are also scala libraries - notably [avro4s](https://github.com/sksamuel/avro4s).
This latter project provides far more capabilities than the example here, especially
if you need to process data from external sources where non-scala field name conventions
are used.

### So why write another tool?
In short, the path of least resistance.
In our company, Optrak, we already have familiarity with Magnolia having written several generators already,
(xml writing and parsing, jdbc record processing, csvs, xlsx, typesafe config etc) and
it looked quicker to do another than to get to grips with a totally new tool.
We are going to be writing and consuming data that is entirely under our control and thus can
managed and we wish to support extended types ([squants units](https://github.com/typelevel/squants)
for example) that would probably require extensions to any external library.

The source code associated with this blog is a subset of the internal library that we have created. This
is to keep down the number of external dependencies.

### The rest of this article
The remainder of this article is split into the following sections:
* [Schema generation](_pages/magnolia-avro/schema.md)
* [Avro Writer](_pages/magnolia-avro/avro-writer.md)
* [Avro Reader](_pages/magnolia-avro/avro-reader.md)
* [Collections](_pages/magnolia-avro/collections.md)
* [Date and time](_pages/magnolia-avro/datetime.md)
* [Serialization](_pages/magnolia-avro/serialization.md)

## Source Code
All source code, including build.sbt can be found
[here](https://github.com/TimPigden/zio-http4s-examples). This blog refers
to the avro-magnolia sub-project.


