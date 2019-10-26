---
layout: default
title: Date and Time
description: Examples of use of magnolia library to generate schema, reader and writer typeclasses for Avro serialization
---

# Date and Time

We make extensive use of java.time types. But Avro has no support for dates and times.

## Schema

For all except ZonedDateTime it is possible to use a Long with milliseconds. Although
precision could be greater for some of the types, millis throughout provided greater consistency
and is adequent for my needs.

```scala
  implicit val instantSchema: SchemaGenerator[Instant] = new SchemaGenerator[Instant] {
    override def generate: JValue = longAvroSchema.generate
  }

  implicit val localDateSchema: SchemaGenerator[LocalDate] = new SchemaGenerator[LocalDate] {
    override def generate: JValue = longAvroSchema.generate
  }

  implicit val localDateTimeSchema: SchemaGenerator[LocalDateTime] = new SchemaGenerator[LocalDateTime] {
    override def generate: JValue = longAvroSchema.generate
  }

  implicit val localTimeSchema: SchemaGenerator[LocalTime] = new SchemaGenerator[LocalTime] {
    override def generate: JValue = longAvroSchema.generate
  }  
```

For ZonedDateTime it seemed easiest to split into a ZoneId plus Instant
```scala
  case class ZoneInstant(zoneIdS: String, instant: Instant) {
    def toZonedDateTime: ZonedDateTime = ZonedDateTime.ofInstant(instant, ZoneId.of(zoneIdS))
  }

  implicit val zonedDateTimeSchema: SchemaGenerator[ZonedDateTime] = new SchemaGenerator[ZonedDateTime] {
    private val zoneInstantSchema = AvroSchemaDerivation.avroSchema[ZoneInstant]
    override def generate: JValue = zoneInstantSchema.generate
  }
  
    lazy val utcId = ZoneId.of("UTC")
```
The utcId declaration is used for localDateTime - see below

## Writer

```scala
  implicit val instantWriter: AvroWriter[Instant] = { (schema, value) =>
    longAvroWriter.write(schema, value.toEpochMilli)
  }

  implicit val localDateWriter: AvroWriter[LocalDate] = { (schema, value) =>
    longAvroWriter.write(schema, value.toEpochDay)
  }
  implicit val localDateTimeWriter: AvroWriter[LocalDateTime] = { (schema, value) =>
    val inst = value.atZone(Utils.utcId).toInstant
    longAvroWriter.write(schema, inst.toEpochMilli)
  }

  implicit val localTimeWriter: AvroWriter[LocalTime] ={ (schema, value) =>
    val asMillis = value.toNanoOfDay / 1000000L
    longAvroWriter.write(schema, asMillis)
  }

  implicit val ZoneIdWriter: AvroWriter[ZoneId] = { (schema, value) =>
    stringAvroWriter.write(schema, value.getId)
  }

  private val zoneInstantWriter = AvroWriterDerivation.avroWriter[ZoneInstant]

  implicit val zonedDateTimeWriter: AvroWriter[ZonedDateTime] = { (schema, value) =>
    val zii = ZoneInstant(value.getZone.getId, value.toInstant)
    zoneInstantWriter.write(schema, zii)
  }
```

## Reader
```scala
  implicit val instantReader: AvroReader[Instant] = { (schema, data) =>
    val instantL = longAvroReader.read(schema, data)
    Instant.ofEpochMilli(instantL)
  }

  implicit val localDateReader: AvroReader[LocalDate] = { (schema, data) =>
    val localDateL = longAvroReader.read(schema, data)
    LocalDate.ofEpochDay(localDateL)
  }
  implicit val localDateTimeReader: AvroReader[LocalDateTime] = { (schema, data) =>
    val instantL = longAvroReader.read(schema, data)
    val inst = Instant.ofEpochMilli(instantL)
    LocalDateTime.ofInstant(inst, Utils.utcId)
  }
  implicit val localTimeReader: AvroReader[LocalTime] = { (schema, data) =>
    val localTimeL = longAvroReader.read(schema, data)
    LocalTime.ofNanoOfDay(localTimeL * 1000000L)
  }

  implicit val zonedDateTimeReader: AvroReader[ZonedDateTime] = new AvroReader[ZonedDateTime] {
    private val zoneInstantReader = AvroReaderDerivation.avroReader[ZoneInstant]

    override def read(schema: Schema, data: AnyRef): ZonedDateTime =
      zoneInstantReader.read(schema, data).toZonedDateTime
  }
```