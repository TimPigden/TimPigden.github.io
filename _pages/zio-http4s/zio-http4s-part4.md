# ZIO - Http4s Part4 - Custom Codecs

Codecs are used to convert Http request and response bodies to and from the
things that are meaningful to your program.

Http4s comes with support for json via circe and some other lower-level codecs.
But it is by no means a complete set for all requirements.

This example shows how to construct a custom codec for use with http4s and Zio. In this
case the data type is *Person* and we're going to encode it in xml.
```scala