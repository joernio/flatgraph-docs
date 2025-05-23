+++
archetype = "home"
title = "Overview"
weight = 1
+++

[flatgraph](https://github.com/joernio/flatgraph) is a fast and memory efficient columnar graph database with a framework to generate domain specific and typesafe traversals. 
Model your domain in flatgraph's schema builder ([examples](https://github.com/joernio/flatgraph/tree/master/test-schemas/src/main/scala/flatgraph/testdomains)) and the code generator will generate all required classes and steps required to traverse your domain in a graph as well persistence. Note: flatgraph is not supposed to be a full ACID compliant database - our design goals are efficiency, minimalism, simplicity and rapid prototyping. 

flatgraph is developed and maintained as the underlying database for the code analysis platform [joern](https://joern.io) as well as [qwiet.ai's](https://qwiet.ai/) proprietary static code analysis tools. Many design decisions were driven by the needs of code analysis, but flatgraph is useful for other domains as well, and since we love open source (and hope to get bug reports and feature contributions back), we decided to share it with the world. 

## Main characteristics / design decisions
* runs locally in a JVM (written in [Scala 3](https://www.scala-lang.org/))
* provides a DSL to model your domain-specific graph schema
* code generator creates a domain specific typesafe query language
* code generator can be used programmatically or as an sbt plugin
* memory and storage efficient: see memory footprint
* very fast traversals across connected nodes

## Non-features: do _not_ use flatgraph if you need...
* transactions, and therefor all ACID properties (atomicity, consistency, isolation, durability)
* fail safety: flatgraph does not create a write-ahead log - this is not a design decision though and could be implemented if needed
* remoting: flatgraph does not provide a client in order to connect to a remote flatgraph instance and/or manage concurrent clients
* clustering: flatgraph does not support multiple instances that coordinate each other
* swapping to disk: your graph will need to fit into the heap at all times

## Glimpse of a simple use case
We have a history in [tinkerpop](https://tinkerpop.apache.org), so allow us to use a very simple graph domain from there to show how to get started with flatgraph. Working code is typically better than prose: the full setup is part of the tests in the [flatgraph repository](https://github.com/joernio/flatgraph).
{{< figure src="/grateful-dead-schema.png" caption="grateful dead sample domain" width="600px">}}

Excerpt from the [GratefulDead domain schema](https://github.com/joernio/flatgraph/blob/44005cf16373dfaf629da8628071ebbfaf02b551/test-schemas/src/main/scala/flatgraph/testdomains/GratefulDead.scala):
```scala
val builder  = new SchemaBuilder("GratefulDead", basePackage = "testdomains.gratefuldead")
val name     = builder.addProperty("name", ValueType.String).mandatory(default = "")
val songType = builder.addProperty("songType", ValueType.String)
val artist   = builder.addNodeType("artist").addProperty(name)
val song     = builder.addNodeType("song").addProperty(name).addProperty(songType)
val sungBy   = builder.addEdgeType("sungBy")
song.addOutEdge(
  sungBy, 
  inNode = artist, 
  cardinalityOut = Cardinality.One, 
  stepNameOut = "sungBy", 
  stepNameIn = "sang")
```

The code generator takes the schema as input and generates [domain specific classes](https://github.com/joernio/flatgraph/tree/44005cf16373dfaf629da8628071ebbfaf02b551/test-schemas-domain-classes/src/main/scala/testdomains/gratefuldead) that can e.g. be used as follows:

```scala
val gratefulDead = GratefulDead.empty

// create a simple graph with two nodes and two edges
val diffGraph    = GratefulDead.newDiffGraphBuilder
val garcia       = NewArtist().name("Garcia")
val boDiddley    = NewArtist().name("Bo_Diddley")
val heyBoDiddley = NewSong().name("HEY BO DIDDLEY").songtype("cover").performances(5)
// n.b. if nodes are referenced in edges, they don't need to be added separately via `diffGraph.addNode`
diffGraph.addEdge(src = heyBoDiddley, dst = boDiddley, Writtenby.Label)
diffGraph.addEdge(src = heyBoDiddley, dst = garcia, Sungby.Label)
DiffGraphApplier.applyDiff(gratefulDead.graph, diffGraph)

// now query it in the typesafe domain-specific query language
gratefulDead.artist.name.sorted shouldBe List("Bo_Diddley", "Garcia")
gratefulDead.artist.name("Garcia").sang.name.l shouldBe List("HEY BO DIDDLEY")
gratefulDead.song.writtenBy.name.l shouldBe List("Bo_Diddley")
```

{{% notice note %}}
By defining your domain in a schema we enable the compiler to help us write domain-specific code, i.e. the compiler ensures that our traversals are valid and we also get domain-specific code completion. Additionally, we ensure that no schema-invalid graph can be constructed. 
{{% /notice %}}
Some examples for what return types the compiler derives, based on the information given in the schema:
* `gratefulDead.artist: Iterator[Artist]`
* `gratefulDead.artist.name: Iterator[String]`
* `gratefulDead.artist.name("Garcia"): Iterator[Artist]` - note that `name(String)` is a filter step
* `gratefulDead.song.performances: Iterator[Int]`
* `gratefulDead.artist.performances` does not compile
* `gratefulDead.song.writtenBy: Iterator[Artist]`
* `gratefulDead.artist.writtenBy` does not compile

If you didn't provide a name for a relationship in the schema definition, there are some alternative auto-generated options like `sungbyOut` and `artistViaSungbyOut`.
The full working test is [here](https://github.com/joernio/flatgraph/blob/92f4cc4b84bf6b8315971128995a75872376dcff/tests/src/test/scala/flatgraph/GratefulDeadTests.scala).

## Memory footprint
flatgraph uses an efficient columnar layout to make most of your available heap. The actual space required depends on your domain, but to get a rough idea we can look at some numbers from the code analysis tool [joern](https://joern.io): for the analysis of the linux networking driver source, joern creates a flatgraph instance with 9m nodes, 88m node properties, 84m edges and 19m edge properties. This requires 3.7G heap, and if we persist it to disk storage it takes up 300MB. 

## Dependencies
We tried to streamline the dependency tree as much as possible: flatgraph-core most notably depends on [zstd-jni](https://github.com/luben/zstd-jni) (fast compression for storage), [Scala 3](https://www.scala-lang.org/) and [ujson](https://com-lihaoyi.github.io/upickle/#uJson). 


# TODOs
* full setup with sbt and codegen plugin: on separate page, link here
* setup example repo and describe here
    * demo node specific starter steps, regex filter steps, number filter steps, boolean filter steps
  * explain concepts of `NewNode` and StoredNode
  * graph modifications all go via the DiffGraph api. Prefer few large DiffGraph applications over many small ones, since the cost of applying a DiffGraph is almost independent of it's size. 
  * DiffGraphs: cost of 
  * link to the tests in the flatgraph repo
  * link to this page in traversal steps
  * link to codepropertygraph
* generic graph traversals (a.k.a. steps)
  * based on the  always double check the result type 
  * copy from joern docs traversal-basics.md
  * describe the difference of steps between Iterator[X] and X
* algorithms: e.g. shortest path
* import/export formats
* logo: 
  * generate one? similar to joern? asked fabs
  * replace / get rid of the relearn logo
* go through TODOs in text above
* bring online
  * link from flatgraph repo
