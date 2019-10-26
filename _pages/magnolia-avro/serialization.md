---
layout: default
title: Serialization
description: Examples of use of magnolia library to generate schema, reader and writer typeclasses for Avro serialization
---

# Serialization

Writer and Reader work with GenericData. We haven't actually turned the data
into binary suitable for writing to file or transmission.

## Format
One of the characteristics of Avro is that the Schema is often included
with the data as a header. This works well for large files of multiple records, but is less
suitable for the many small records you might get in Kafka.

Here we use the format with the schema included.

## Writing

```scala
  def writeBinary[T](ts: Iterable[T])
                    (dfCreate: (DataFileWriter[GenericRecord], Schema) => Unit)
                    (implicit schemaGenerator: SchemaGenerator[T], avroWriter: AvroWriter[T]): Unit = {
    val asJson = schemaGenerator.generate
    val schema = AvroCompiler.compile(asJson)

    val datumWriter = new GenericDatumWriter[GenericRecord](schema)
    val dataFileWriter = new DataFileWriter(datumWriter)
    dfCreate(dataFileWriter, schema)
    ts.foreach { t =>
      val gr = avroWriter.write(schema, t).asInstanceOf[GenericRecord]
      dataFileWriter.append(gr.asInstanceOf[GenericRecord])
    }
    dataFileWriter.flush()
    dataFileWriter.close()
  }

  def writeBinaryFile[T](ts: Iterable[T], file: File)
                        (implicit schemaGenerator: SchemaGenerator[T], avroWriter: AvroWriter[T]): Unit =
    writeBinary(ts){ (dfw, schema) => dfw.create(schema, file)}
```
The above code is straight-forward. The main complication is that the DataFileWriter has different
create methods so *dfCreate* it is passed in as a parameter.
In this case we use the method that takes a File but others are possible.
See the Avro documentation.

The above system is set up to write multiple records of the same schema.

## Reading

```scala
  def readBinaryFile[T](file: File)(implicit schemaGenerator: SchemaGenerator[T], avroReader: AvroReader[T]): List[T] = {
    val schema = AvroCompiler.compile(schemaGenerator.generate)
    val datumReader = new GenericDatumReader[AnyRef](schema)
    val dataFileReader = new DataFileReader(file, datumReader)
    val record = new GenericData.Record(schema)
    val out = ListBuffer.empty[T]
    while (dataFileReader.hasNext) {
      dataFileReader.next(record)
      val t = avroReader.read(schema, record)
      out.append(t)
    }
    dataFileReader.close()
    out.toList
  }
```

## Usage
The following test method illustrates usage:
```scala
  def writeReadBackFile[T](protos: List[T],
                           tFile: File)
                          (implicit schemaGenerator: SchemaGenerator[T],
                           reader: AvroReader[T],
                           writer: AvroWriter[T]): Assertion = {
    AvroSerialize.writeBinaryFile[T](protos, tFile)
    val readBack = AvroSerialize.readBinaryFile[T](tFile)
    readBack should === (protos)
  }
  
  case class IntSet(l: Set[Int])
  val iSet1 = IntSet(Set(1,2))
  val iSet0 = IntSet(Set.empty[Int])
  
  "as file" in {
    writeReadBackFile(List(iSet1, iSet0), new File("data/avro/testout.avro"))
  }
```

## Schema Registry
Another possibility is to use
a schema registry, such as the one provided by [Confluent](https://docs.confluent.io/current/schema-registry/index.html). This is based on a persistent, write-once,
store of schemas, each of which is indexed by an integer which is generated
when the schema is registered (after first checking it does not match
a pre-existing schema). Confluent provides the necessary libraries for
accessing the registry and pre-pending the schema index to the binary file.




