---
layout: default
title: AvroReader - Reading Avro Binary data
description: Examples of use of magnolia library to generate schema, reader and writer typeclasses for Avro serialization
---

# AvroReader - Reading Binary Data

AvroReader takes the GenericData created by AvroWriter and turns it back into our
case class or whatever.

## Typeclass

Not surprisingly, the typeclass is essentially the reverse of AvroWriter
```scala
trait AvroReader[A] {
  def read(schema: Schema, data: AnyRef): A
}
```

We need the schema to tell us how to unpack the data and the data itself
is AnyRef because at the bottom level we have to deal with primitive data
types.

Again the example for Int
```scala
implicit val intAvroReader: AvroReader[Int] = { (_, ref) => ref match {
    case v: Integer => v.intValue
  }}
```
We know it's a java.lang.Integer - so we do not need a more comprehensive match.

## The Generator
```scala
object AvroReaderDerivation {
  type Typeclass[T] = AvroReader[T]

  def combine[T](ctx: CaseClass[AvroReader, T]): AvroReader[T] = { (schema, data) =>
    if (ctx.isObject)
      ctx.rawConstruct(Seq.empty)
    else data match {
      case r: GenericRecord =>
        val fields = ctx.parameters.map { param =>
          val thisSchema = schema.getField(param.label).schema()
          val fieldObj = r.get(param.label)
          param.typeclass.read(thisSchema, fieldObj)
        }

        ctx.rawConstruct(fields)
      case x => throw new IllegalArgumentException(s"avro reader wrong type $x")
    }
  }

  def dispatch[T](ctx: SealedTrait[AvroReader, T]): AvroReader[T] = { (schema, data) =>
    data match {
      case gd: GenericData.Record =>
        val thisSchema = gd.getSchema
        val subtype = ctx.subtypes.find(_.typeName.short == thisSchema.getName).get
        subtype.typeclass.read(thisSchema, data)
    }

  }

  implicit def avroReader[T]: AvroReader[T] = macro Magnolia.gen[T]
}
```

### combine
The first 2 lines of combine deal with the special case of a scala *case object* - which has no parameters.
We then map over the fields invoking the typeclass with the child schemas and data.

The *ctx.rawConstruct(fields)* is the important part - this will create a case class from the fields.

### dispatch
For dispatch we obtain the schema for the data in question. We then find our
subtype with a matching name and get the data.

# Putting it together

The following is from the test code for Simple (using ScalaTest)
```scala
import AvroSchemaDerivation._
import AvroWriterDerivation._
import AvroReaderDerivation._
object TestSchema {
  case class Simple(i: Int, d: Double)
  val simpleProto = Simple(1, 2.0)
  
  def writeReadBackList[T](protos: List[T])
                      (implicit schemaGenerator: SchemaGenerator[T],
                       reader: AvroReader[T],
                       writer: AvroWriter[T]): Assertion = {
    val asJson = schemaGenerator.generate
    println(pretty(asJson))
    val schema = AvroCompiler.compile(asJson)
    val andBack = schema.toString(false)
    println(s"schema\n$andBack")
    val records = protos.map{ p =>
      writer.write(schema, p)
    }
    val res = records.map { r => reader.read(schema, r ) }
    println(s"read is $res")
    res should === (protos)
  }
}
class TestSchema extends WordSpecLike with Matchers {  
  "simple" in {
    writeReadBackList(List(simpleProto))
  }
} 
```
All we need to invoke the method is have the appropriate Derivation methods in scope.


