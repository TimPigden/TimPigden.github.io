---
layout: default
title: AvroWriter - Generating Avro Binary data
description: Examples of use of magnolia library to generate schema, reader and writer typeclasses for Avro serialization
---
# AvroWriter

In the [previous section](_pages/magnolia-avro/schema.md) we generated schemas
for Avro. In this section we're going to start to write out our data. I say "start"
because we will use a 2-step process. The first is to generate Avro GenericData
objects, the second is to actually serialize those to binary. In this section
I'll cover the first step.

GenericData is not the only way to work with data in Avro. There is also SpecificRecord. But GenericData.Record
works for our purposes and so is the only method addressed here.

## The Typeclass - AvroWriter

```scala
trait AvroWriter[A]  {
  def write(schema: Schema, value: A): AnyRef
}
```

The schema in this case is the schema for A. As noted previously, it's
"schemas all the way down", so at each level, including primitives, we
have a schema - though we don't always need it.

The *value* is simply the value of our current data.

The return type is AnyRef - essentially a java Object. This is how Avro
GenericData works - higher-level structures have a GenericData subtype, but at
the bottom level we're storing things like java.lang.Integer. And here
is the typeclass for Int
```scala
  implicit val intAvroWriter: AvroWriter[Int] =  { (_, value) => value: Integer   }
```
As an aside - for those not familiar with this method of defining typeclasses, this is equivalent
to
```scala
  implicit val intAvroWriter: AvroWriter[Int] =  new AvroWriter[Int] {
    def write(schema: Schema, value: Int): AnyRef = value: Integer   
  }
```
It's just that we have a SAM (Single Abstract Method) and so we can use an abbreviated syntax.

So for Int, all our method does is convert the Int to a scala.lang.Integer

## The Writer
```scala
object AvroWriterDerivation {
  type Typeclass[T] = AvroWriter[T]

  def combine[T](ctx: CaseClass[AvroWriter, T]): AvroWriter[T] =
    new AvroWriter[T] {
      override def write(schema: Schema, value: T): AnyRef = {
        val record = new GenericData.Record(schema)
        ctx.parameters.foreach { param =>
          val thisSchema = schema.getField(param.label).schema()
          val fieldVal = param.dereference(value)
          val res = param.typeclass.write(thisSchema, fieldVal)
          record.put(param.label, res)
        }
        record
      }
    }

  def dispatch[T](ctx: SealedTrait[AvroWriter, T]): AvroWriter[T] = { (schema, value) =>
    ctx.dispatch(value) { sub =>
      val thisSchema = schema.getTypes.asScala.find(_.getName == sub.typeName.short).get
      sub.typeclass.write(thisSchema, sub.cast(value))
    }
  }

  implicit def avroWriter[T]: AvroWriter[T] = macro Magnolia.gen[T]

}
```
### combine
The first thing the combine function does is to create a GenericData.Record. This will
hold all our field data. It works pretty much like a Map of field name to data.
We then iterate over the parameters of case class, getting schema and field data,
create a new AnyRef by calling *param.typeclass.write* and put the data into
the record using the param label.
Finally, we return the record.

### dispatch
For dispatch we find the subtype that we actually have, using ctx.dispatch, look up it's schema
(note the Option.get - we're confident it exists) and then return the value for that typeclass.

For both combine and dispatch you will see we are using a variety of magnolia functions. Please refer
to the magnolia documentation for further explanation.

