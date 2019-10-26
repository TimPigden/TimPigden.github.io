---
layout: default
title: Avro Schema Generation
description: Examples of use of magnolia library to generate schema, reader and writer typeclasses for Avro serialization
---

# Schema Generation

Avro schemas are usually written in json and then compiled into an internal format.

For our first example, we will create a schema for the following case class:
```scala
  case class Simple(i: Int, d: Double)
```
The json version of the schema will look like this (at least when pretty-printed)
```json
{
  "type" : "record",
  "name" : "Simple",
  "namespace" : "tsp.avro.TestSchema",
  "fields" : [ {
    "name" : "i",
    "type" : "int"
  }, {
    "name" : "d",
    "type" : "double"
  } ]
}
```
The first point to note is that it's schemas all the way down. A record contains
fields that have a type that can be a primitive (int, double, etc.) or another record,
or an array or whatever. The schema definition documentation can be found
[here](http://avro.apache.org/docs/current/spec.html#schemas).

We need to generate json. There are a variety of json libraries for scala out there.
The one I'm familiar with is [json4s](https://github.com/json4s/json4s) and so that's
what will be used in this code.

## The Typeclass

The typeclass is SchemaGenerator
```scala
trait SchemaGenerator[A]  {
  def generate: JValue
}
```

It has a single method *generate* which produces a Jvalue corresponding to the type
of the *A*. Here's the generator for Int
```scala
  implicit def intAvroSchema: SchemaGenerator[Int] = new SchemaGenerator[Int] {
    override def generate = JString("int")
  }
```
Note that figuring out what the typeclass should do is one of the critical
parts of using magnolia. In this case it was fairly clear - we wanted the type. The
other stuff can be left to the Derivation object.

## Magnolia Schema Derivation

I will first list the entire derivation object and then look at the bits individually:
```scala
object AvroSchemaDerivation {
  type Typeclass[T] = SchemaGenerator[T]

  def combine[T](ctx: CaseClass[SchemaGenerator, T]): SchemaGenerator[T] = new SchemaGenerator[T] {
    override def generate: JValue = {
      val fields = ctx.parameters.map { param =>
        val res = param.typeclass.generate
        JObject("name" -> JString(param.label),
            "type" -> res)
      }.toList
      JObject(
        "type" -> JString("record"),
        "name" -> JString(ctx.typeName.short),
        "namespace" -> JString(ctx.typeName.owner),
        "fields" -> JArray(fields)
      )
    }
  }

  def dispatch[T](ctx: SealedTrait[SchemaGenerator, T]): SchemaGenerator[T] = new SchemaGenerator[T] {
    override def generate: JValue = {
      val children = ctx.subtypes.map { st =>
        st.typeclass.generate
      }
      JArray(children.toList)
    }
  }

  implicit def avroSchema[T]: SchemaGenerator[T] = macro Magnolia.gen[T]

}
```

Each Derivation object has 4 essential ingredients.
* the Typeclass definition
* the combine function
* the dispatch function
* the implicit call to the macro processing

The Typeclass definition is just a declaration of our typeclass - in this case
*SchemaGenerator*

### combine
The combine function creates an automatic typeclass for T. T is a case
class (the macro takes care of this part).
```scala
  def combine[T](ctx: CaseClass[SchemaGenerator, T]): SchemaGenerator[T] = new SchemaGenerator[T] {
    override def generate: JValue = {
      val fields = ctx.parameters.map { param =>
        val res = param.typeclass.generate
        JObject("name" -> JString(param.label),
            "type" -> res)
      }.toList
      JObject(
        "type" -> JString("record"),
        "name" -> JString(ctx.typeName.short),
        "namespace" -> JString(ctx.typeName.owner),
        "fields" -> JArray(fields)
      )
    }
  }
```

The first job is to generate the sub-schemas for all the fields of the case
class. To do this we map over the ctx.parameters - which gives us a parameter
per field and call generate on each field (*res*) to give us the JValue of
the type. For *Simple* field *i* this will be the text "int". For the full text
we create a JObject with name - name text is taken from param.label and
type which will be set to "int". The JObject is thus the JValue of
```json
{
    "name" : "i",
    "type" : "int"
  }
```
We then combine this with data for the case class itself. Note that we are
using *ctx.typeName.owner* to auto-generate our namespace - which is therefore
the package and any containing object of our case class.

### dispatch

Dispatch is used for sealed traits - or *coproducts*. Avro represents these as a
Union type. The following sealed trait *Stuff* shows this, *WithStuff* being
an enclosing case class.

```scala
  sealed trait Stuff
  case object AStuff extends Stuff
  case object BStuff extends Stuff
  case class CStuff(j: Int) extends Stuff

  case class WithStuff(i: Int, stuff1: Stuff)
```
This is used to generate the following schema
```json
{
  "type" : "record",
  "name" : "WithStuff",
  "namespace" : "tsp.avro.TestSchema",
  "fields" : [ {
    "name" : "i",
    "type" : "int"
  }, {
    "name" : "stuff1",
    "type" : [ 
        {
          "type" : "record",
          "name" : "AStuff",
          "namespace" : "tsp.avro.TestSchema",
          "fields" : [ ]
        }, {
          "type" : "record",
          "name" : "CStuff",
          "namespace" : "tsp.avro.TestSchema",
          "fields" : [ {
            "name" : "j",
            "type" : "int"
          } ]
        }, {
          "type" : "record",
          "name" : "BStuff",
          "namespace" : "tsp.avro.TestSchema",
          "fields" : [ ]
        } 
    ]
  } ]
}
```
Note how the union type is simply represented by a json array of the types
that make it up.

Consequently our dispatch method is quite simple:
```scala
  def dispatch[T](ctx: SealedTrait[SchemaGenerator, T]): SchemaGenerator[T] = new SchemaGenerator[T] {
    override def generate: JValue = {
      val children = ctx.subtypes.map { st =>
        st.typeclass.generate
      }
      JArray(children.toList)
    }
  }
```
We simply map over all subtypes (of Stuff in this case) and generate JValues
for each, wrapping the whole in a Json array.

## Automatic derivation
So how do we actually use it?

We can simply call something like
```scala
  val simpleGenerator = AvroSchemaDerivation.avrosSchema[Simple]
```
or generate it on the fly
```scala
import AvroSchemaDerivation._

def needsSchema[T](t: T)(implicit schemaGen: SchemaGenerator[T]) = ...

needsSchema(mySimple)
```

## Compiling the Schema

Compiling the schema involves converting the JValue to a string and using
the Schema.parser to create the actual schema.
```scala
import org.apache.avro.Schema
object AvroCompiler {
  def compile(jv: JValue):Schema = {
    new Schema.Parser().parse(compact(jv))
  }
}
```
here *compact* is the Json4s function to convert a JValue to a compact json string.

In this section we have covered generation of case classes of sealed traits and
primitives. Collections and arrays are dealt with in [collections](_pages/magnolia-avro/collections.md).

## It doesn't have to be a case class
It's worth noting that there is no obligation for the root of your Schema to be a case class. You can
directly create a schema for other types and use it to generate the required writer and readers.