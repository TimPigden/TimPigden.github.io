---
layout: default
title: Collections - dealing with Collections in Avro Data
description: Examples of use of magnolia library to generate schema, reader and writer typeclasses for Avro serialization
---

# Option and Collections

## Option
In Avro Option is best represented as a Union type of *null* and the data.

```scala
  implicit def optionSchema[T](implicit tGenerator: SchemaGenerator[T]): SchemaGenerator[Option[T]] = new SchemaGenerator[Option[T]] {
    override def generate: JValue = JArray(List("null", tGenerator.generate))
  }

  implicit def optionWriter[T](implicit tWriter: AvroWriter[T]): AvroWriter[Option[T]] = { (schema, value) =>
    value match {
      case Some(t) => tWriter.write(schema, t)
      case None => null
    }
  }
  
  implicit def optionReader[T](implicit tReader: AvroReader[T]): AvroReader[Option[T]] = { (schema, data) =>
    Option(data).map { d => tReader.read(schema, d) }
  }
```
In this case "null" is an Avro primitive and is used for any nullable data. We use the Option(t) constructor
to convert it back into the safer None or Some(t)

## Iterable Collections

Examples - for List
```scala
  def iterableSchema[T](implicit tGenerator: SchemaGenerator[T]): SchemaGenerator[Iterable[T]] = new SchemaGenerator[Iterable[T]] {
    override def generate: JValue =
      ("type" -> "array") ~
        ("items" -> tGenerator.generate)
  }

  implicit def listSchema[T](implicit tGenerator: SchemaGenerator[T]): SchemaGenerator[List[T]] = new SchemaGenerator[List[T]] {
    override def generate: JValue = iterableSchema[T].generate
  }

  def iterableAvroWriter[T](implicit tWriter: AvroWriter[T]): AvroWriter[Iterable[T]] = { (schema, value) =>
    val elType = schema.getElementType
    val children = value.toList.map { v => tWriter.write(elType, v)}
    new GenericData.Array(schema, children.asJava)
  }

  implicit def listAvroWriter[T](implicit tWriter: AvroWriter[T]): AvroWriter[List[T]] = new AvroWriter[List[T]] {
    private val iterable = iterableAvroWriter[T]
    override def write(schema: Schema, value: List[T]): AnyRef = iterable.write(schema, value)
  }

  def iterableAvroReader[T](implicit tReader: AvroReader[T]): AvroReader[Iterable[T]] = { (schema, ref) =>
    val elType = schema.getElementType
    ref match {
      case ar: GenericData.Array[_] =>
        ar.asScala.map {
          case ref: AnyRef => tReader.read(elType, ref)
        }
    }
  }

  implicit def listAvroReader[T](implicit tReader: AvroReader[T]): AvroReader[List[T]] = new AvroReader[List[T]] {
    private val iterableReader = iterableAvroReader[T]
    override def read(schema: Schema, data: AnyRef): List[T] = iterableReader.read(schema, data).toList
  }
```
Avro has a GenericData.Array that can be used to store any iterable. Note that
internally it uses java Collection type. So ironically you can't actually use
it for Array - since that is not a Collection.

## Arrays of Primitive

Arrays of object are relatively uncommon in scala. I've not provided any
implementation for them. However, arrays of primitive are useful for binary
storage of large amounts of like data (I use them a lot in mathematical optimisation code).

The Avro GenericData.Array type cannot be used for this purpose as it works for boxed collections only. However,
there is an avro primitive type "bytes" which is the low-level storage of an array of bytes. In fact
Avro expects a nio ByteBuffer.

```scala
  implicit def floatArraySchema[T <: Float]: SchemaGenerator[Array[T]] = new SchemaGenerator[Array[T]] {
    override def generate: JValue = JString("bytes")
  }
  
  implicit def floatArrayAvroWriter: AvroWriter[Array[Float]] = { (_, value) =>
    val bb = ByteBuffer.allocate(value.length * 4)
    value.indices.foreach { i => bb.putFloat(value(i))}
    bb.rewind()
  }

  implicit val floatArrayAvroReader: AvroReader[Array[Float]] = { (_, ref) => ref match {
    case bb: ByteBuffer =>
      val flts = bb.asFloatBuffer()
      val lim = flts.limit()
      val out = Array.ofDim[Float](lim)
      0.until(lim).foreach( i => out(i) = flts.get(i))
      out
    }
  }
```

## String-keyed Maps
Avro provides direct support for String-keyed maps.
```scala
  def stringMapSchema[K <: String, T](implicit tGenerator: SchemaGenerator[T]): SchemaGenerator[Map[K, T]] =
    new SchemaGenerator[Map[K, T]] {
      override def generate: JValue = ("type" -> "map") ~ ("values" -> tGenerator.generate)
    }
    
  def stringMapAvroWriter[K <: String, T](implicit tWriter: AvroWriter[T]): AvroWriter[Map[K, T]] = { (schema, value) =>
    val elType = schema.getValueType
    val children: java.util.Map[String, AnyRef] = value.map { p =>
      val s: String = p._1
      val v = p._2
      s -> tWriter.write(elType, v)
    }.asJava
    children
  }
    
  def stringKMapAvroReader[K <: String, T](implicit tReader: AvroReader[T], toK: String => K): AvroReader[Map[K, T]] = { (schema, ref) =>
    val elType = schema.getValueType
    ref match {
      case ar: java.util.Map[_, _] =>
        ar.asScala.map { p =>
          val t = p._2 match {
            case ref: AnyRef => tReader.read(elType, ref)
          }
          val k = p._1 match {
            case s: String => toK(s)
          }
          k -> t
        }.toMap
    }
  }

  def stringMapAvroReader[T](implicit tReader: AvroReader[T]): AvroReader[Map[String, T]] = new AvroReader[Map[String, T]] {
    private val internal = stringKMapAvroReader[String, T](tReader, x => x)

    override def read(schema: Schema, data: AnyRef): Map[String, T] = internal.read(schema, data)
  }
    
```
The slightly convoluted reader is to deal with T <: String and allow correct
generation of the subtype. We use it for tagged string
types (see this [blog article](http://www.vlachjosef.com/tagged-types-introduction/) for an explanation.

## Other maps
Other maps can be dealt with as an iterable of (key, value).

```scala
  def kvMapSchema[K, T](implicit kGenerator: SchemaGenerator[K], tGenerator: SchemaGenerator[T]): SchemaGenerator[Map[K, T]] =
    new SchemaGenerator[Map[K, T]] {
      private val tupleGenerator = AvroSchemaDerivation.avroSchema[(K, T)]
      private val tupleIterator = iterableSchema(tupleGenerator)

      override def generate: JValue = tupleIterator.generate
    }
    
  def kvMapAvroWriter[K, T](implicit kWriter: AvroWriter[K], tWriter: AvroWriter[T]): AvroWriter[Map[K, T]] = new AvroWriter[Map[K, T]] {
    private val tupWriter = AvroWriterDerivation.avroWriter[(K, T)]
    private val itWriter = iterableAvroWriter(tupWriter)

    override def write(schema: Schema, value: Map[K, T]): AnyRef = itWriter.write(schema, value.toList)
  }

  def kvMapAvroReader[K, T](implicit kReader: AvroReader[K], tReader: AvroReader[T]): AvroReader[Map[K, T]] = new AvroReader[Map[K, T]] {
    private val tupReader = AvroReaderDerivation.avroReader[(K, T)]
    private val itReader = iterableAvroReader(tupReader)

    override def read(schema: Schema, ref: AnyRef): Map[K, T] = itReader.read(schema, ref).toMap
  }
```



