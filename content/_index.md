+++
archetype = "home"
title = "flatgraph"
+++

[flatgraph](https://github.com/joernio/flatgraph) is a fast and memory efficient columnar graph database with support for domain specific and typesafe traversal language. Written in [Scala 3](https://www.scala-lang.org/).

flatgraph was developed as the underlying database for the code analysis platform [joern](https://joern.io) as well as [qwiet.ai's](https://qwiet.ai/) proprietary static code analysis tools. Many design decisions were driven by the needs of code analysis, however we are certain that flatgraph is useful for other domains as well, and we've tried our best to make it easy to give it a go. 

## Main characteristics / design decisions
* flatgraph runs locally in a JVM
* provides a DSL to model your domain-specific graph schema
* code generator creates a domain specific typesafe query language
* code generator can be used programmatically or as an sbt plugin
* memory and storage efficient: see memory footprint
* very fast traversals across connected nodes

## Non-features: do _not_ use flatgraph if you need...
* transactions, and therefor all ACID properties (atomicity, consistency, isolation, durability)
* fail safety: flatgraph does not create a write-ahead log - this is not a design decision though and could be implemented if needed
* remoting: flatgraph does not provide a client in order to connect to a remote flatgraph instance and/or manage concurrent clients
* swapping to disk: your graph will need to fit into the heap at all times

## Glimpse of a simple use case
I have a history in [tinkerpop](https://tinkerpop.apache.org), so allow me to take a very simple graph domain from there to show how to get started with flatgraph. Working code is often better than prose: and the full setup is part of the tests in the [flatgraph repository](https://github.com/joernio/flatgraph).
{{< figure src="/grateful-dead-schema.png" caption="grateful dead sample domain">}}

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
The code generator takes the schema as input and generates [domain specific classes](https://github.com/joernio/flatgraph/tree/44005cf16373dfaf629da8628071ebbfaf02b551/test-schemas-domain-classes/src/main/scala/testdomains/gratefuldead) that can e.g. be used like this:
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
Note that the properties (e.g. `name` and `performances`) have the respective types as defined in the schema, and relationships between nodes have named steps, e.g. `sang`. If you didn't provide a name for a relationship, there are some alternative auto-generated options like `sungbyOut` and `artistViaSungbyOut`.
The full working test is [here](https://github.com/joernio/flatgraph/blob/92f4cc4b84bf6b8315971128995a75872376dcff/tests/src/test/scala/flatgraph/GratefulDeadTests.scala).

## Memory footprint
flatgraph uses an efficient columnar layout to make most of your available heap. The actual space required depends on your domain, but to get a rough idea we can look at some numbers from the code analysis tool [joern](https://joern.io): for the analysis of the linux networking driver source, joern creates a flatgraph instance with 9m nodes, 88m node properties, 84m edges and 19m edge properties. This requires 3.7G heap, and if we persist it to disk storage it takes up 300MB. 

# TODOs
* full setup with sbt and codegen plugin: on separate page, link here
  * node specific starter steps, regex filter steps, number filter steps, boolean filter steps
  * link to this page in traversal steps
* graph traversals (a.k.a. steps)
  * start with a domain setup and the domain specific steps
  * copy from joern docs traversal-basics.md
  * l/toSet/toSeq
  * out/in/label
  * nodeCount
  * groupCount
  * size
  * collectAll / collect
  * filter/filterNot
  * where/whereNot
  * copy some basics from joern
  * repeat and friends
  * path tracking
  * go through the remainder of the flatgraph api
* algorithms: e.g. shortest path
* import/export formats
* benchmarks: copy from readme
* logo: 
  * generate one? similar to joern? asked fabs
  * replace / get rid of the relearn logo
* go through TODOs in text above
* bring online
  * host on github pages?
  * configure url in hugo.toml
  * link from flatgraph repo
