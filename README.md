## Jawn

"Jawn is for parsing jay-sawn."

### Origin

The term "jawn" comes from the Philadelphia area. It conveys about as
much information as "thing" does. I chose the name because I had moved
to Montreal so I was remembering Philly fondly. Also, there isn't a
better way to describe objects encoded in JSON than "things". Finally,
we get a catchy slogan.

Jawn was designed to parse JSON into an AST as quickly as possible.

### Overview

Jawn consists of three parts:

1. A fast, generic JSON parser
2. A small, somewhat anemic AST
3. Support packages which parse to third-party ASTs

Currently Jawn is competitive with the fastest Java JSON libraries
(GSON and Jackson) and in the author's benchmarks it often wins. It
seems to be faster than any other Scala parser that exists (as of July
2014).

Given the plethora of really nice JSON libraries for Scala, the
expectation is that you are here for (1) and (3) not (2).

### Quick Start

Jawn supports Scala 2.10 and 2.11. Here's a `build.sbt` snippet that
shows you how to depend on Jawn for your project:

```scala
// required for all uses
resolvers += "bintray/non" at "http://dl.bintray.com/non/maven"

// use this if you just want jawn's parser, and will implement your own facade
libraryDependencies += "org.jsawn" %% "jawn-parser" % "0.5.0"

// use this if you want to use jawn's parser and ast
libraryDependencies += "org.jsawn" %% "jawn-ast" % "0.5.0"
```

If you want to use Jawn's fast parser with another project's AST, see
the "Supporting external ASTs with Jawn" section. For example, with
Spray you would say:

```scala
resolvers += "bintray/non" at "http://dl.bintray.com/non/maven"

libraryDependencies += "org.jsawn" %% "support-spray % "0.5.0"
```

### Parsing

Jawn's parser is both fast and relatively featureful. Assuming you
want to get back an AST of type `J` and you have a `Facade[J]`
defined, you can use the following `parse` signatures:

```scala
Parser.parseUnsafe[J](String) → J
Parser.parseFromString[J](String) → Try[J]
Parser.parsefromPath[J](String) → Try[J]
Parser.parseFromFile[J](File) → Try[J]
Parser.parseFromByteBuffer[J](ByteBuffer) → Try[J]
```

Jawn also supports asynchronous parsing, which allows users to feed
the parser with data as it is available. There are three modes:

* `SingleValue` waits to return a single `J` value once parsing is done.
* `UnwrapArray` if the top-level element is an array, return values as they become available.
* `ValueStream` parser one-or-more json values separated by whitespace

Here's an example:

```scala
import jawn.ast
import jawn.AsyncParser

val p = ast.JParser.async(mode = AsyncParser.UnwrapArray)

def chunks: Stream[String] = ...
def sink(j: ast.JValue): Unit = ...

def loop(st: Stream[String]): Either[ParseException, Unit] =
  st match {
    case Stream.End => p.finish().fold(e => e, js => js.foreach(sink))
    case s #:: tail => p.absorb(s).fold(e => e, { js => js.foreach(sink); loop(tail) })
  }
  
loop(chunks)
```

You can also call `jawn.Parser.async[J]` to use async parsing with an
arbitrary data type.

### Supporting external ASTs with Jawn

Jawn currently supports three external ASTs directly:

 * Argonaut (6.0.4)
 * Rojoma (2.4.3)
 * Spray (1.2.6)

Each of these subprojects provides a `Parser` object (an instance of
`SupportParser[J]`) that is parameterized on the given project's
AST (`J`). The following methods are available:

```scala
Parser.parseUnsafe(String) → J
Parser.parseFromString(String) → Try[J]
Parser.parsefromPath(String) → Try[J]
Parser.parseFromFile(File) → Try[J]
Parser.parseFromByteBuffer(ByteBuffer) → Try[J]
```
  
These methods parallel those provided by `jawn.Parser`.

For the following snippets, `XYZ` is one of (`argonaut`, `rojoma`, or `spray`):

This is how you would include the subproject in build.sbt:

```scala
resolvers += "bintray/non" at "http://dl.bintray.com/non/maven"

libraryDependencies += "org.jsawn" %% "XYZ-support" % "0.5.0"
```

This is an example of how you might use the parser into your code:

```scala
import jawn.support.XYZ.Parser

val myResult = Parser.parseFromString(myString)
```

### Do-It-Yourself Parsing

Jawn supports building any JSON AST you need via type classes. You
benefit from Jawn's fast parser while still using your favorite Scala
JSON library. This mechanism is also what allows Jawn to provide
"support" for other libraries' ASTs.

To include Jawn's parser in your project, add the following
snippet to your `build.sbt` file:

```scala
resolvers += "bintray/non" at "http://dl.bintray.com/non/maven"

libraryDependencies += "jawn" %% "jawn-parser" % "0.5.0"
```

To support your AST of choice, you'll want to define a
`jawn.Facade[J]` instance, where the `J` type parameter represents the
base of your JSON AST. For example, here's a facade that supports
Spray:

```scala
import spray.json._
object Spray extends SimpleFacade[JsValue] {
  def jnull() = JsNull
  def jfalse() = JsFalse
  def jtrue() = JsTrue
  def jnum(s: String) = JsNumber(s)
  def jint(s: String) = JsNumber(s)
  def jstring(s: String) = JsString(s)
  def jarray(vs: List[JsValue]) = JsArray(vs)
  def jobject(vs: Map[String, JsValue]) = JsObject(vs)
}
```

Most ASTs will be easy to define using the `SimpleFacade` or
`MutableFacade` traits. However, if an ASTs object or array instances
do more than just wrap a Scala collection, it may be necessary to
extend `Facade` directly.

### Examples

Jawn can parse JSON from many different sources:

 * `parseFromString(data: String)`
 * `parseFromFile(file: File)`
 * `parseFromPath(path: String)`
 * `parseFromByteBuffer(bb: ByteBuffer)`

Parsing returns `Either[Exception, JValue]`.

### Dependencies

*jawn-parser* has no dependencies other than Scala itself.

*jawn-ast* depends on [Spire](http://github.com/non/spire) in order to
 provide type class instances.

The various support projects (e.g. *argonaut-support*) depend on the
library they are supporting.

### Profiling

Jawn provides benchmarks to help compare various JSON parsers on a
wide range of input files. You can run the benchmarks from SBT with:

```
> benchmark/run
```

Any JSON files you put in `benchmark/src/main/resources` will be
included in the ad-hoc benchmark. There is a Python script I've used
to generate random JSON data called `randjson.py` which is a bit
quirky but does seem to work. I test on this random JSON as well as
data from projects I've worked on.

(I also test with larger data sets (100-600M) but for obvious reasons
I don't distribute this JSON in the project.)

Libraries currently being tested in order of average speed on tests
I've seen:

 * jawn
 * gson
 * play
 * jackson
 * json4s-jackson
 * rojoma
 * argonaut
 * smart-json
 * json4s-native
 * spray
 * lift-json (broken)

Of course, your mileage may vary, and these results do vary somewhat
based on file size, file structure, etc. Also note that Jackson
actually powers many of these libraries (including play, which
sometimes comes out faster than the explicit jackson test for reasons
I don't understand.)

I have tried to understand the libraries well enough to write the most
optimal code for loading a file (given a path) and parsing it to a
simple JSON AST.  Pull requests to update versions and improve usage
are welcome.

### Disclaimers

Jawn only supports UTF-8. This might change in the future, but for now
that's the target case. If you need full-featured support for
character encodings I imagine something like Jackson or Gson will work
better.

The library is still very immature so I'm sure there are some bugs. No
liability or warranty is implied or granted. This project was
initially intended as a proof-of-concept for the underlying design.

### Copyright and License

All code is available to you under the MIT license, available at
http://opensource.org/licenses/mit-license.php.

Copyright Erik Osheim, 2012-2014.
