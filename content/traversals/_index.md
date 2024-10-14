+++
title = "Traversal steps"
weight = 3
+++

The most important basic traversal steps will be the ones generated for your domain as highlighted in [glimpse-of-a-simple-use-case](../_index.html#glimpse-of-a-simple-use-case).

In addition to the generated domain-specific steps based on your schema, there's some basic traversal steps that you can use generically across domains, e.g. to traverse from a node (or an `Iterator[Node]`) to their neighbors, lookup their properties etc. 
There are also more advanced steps like `repeat` and advanced features like path tracking which will be described further below. 

{{% notice tip %}}
flatgraph traversals are based on Scala's Iterator, so you can also use all regular [collection methods](https://docs.scala-lang.org/scala3/book/collections-methods.html). If you want to begin a traversal from a given node, the `.start` method will wrap that node in a traversal making the traversal steps available.
{{% /notice %}}

## Step Types

The various traversal queries can be divided into a number of types: Filter, Map, Side Effect, and Terminal.

### Filter Steps

_Filter Steps_ are atomic traversals that filter nodes according to given criteria. The most common filter step is aptly-named `filter`,  which continues the traversal in the step it suffixes for all nodes which pass its criterion. Its criterion is represented by a lambda function which has access to the node of the previous step and returns a boolean.  Continuing with the previous example, let us execute a query which returns all `METHOD` nodes of the Code Property Graph for [`X42`](https://github.com/ShiftLeftSecurity/x42.git), but only if their `IS_EXTERNAL` property is set to `false`:

```java
joern> cpg.method.filter(_.isExternal == false).name.toList 
res11: List[String] = List("main")
```

{{% notice tip %}}
A note on Scala lambda functions:
In the example above, we used the lambda function `_.isExternal == false` as the predicate for the filter.
The `_` is simply syntactic sugar referring to the parameter of the function, so this could be rewritten
as `method => method.isExternal == false`.
{{% /notice %}}

Dissecting this query, we have `cpg` as the root object, a node-type step `method` which returns all nodes of type `METHOD`, a filter step `where(_.isExternal == false)` which continues the traversal only for nodes which have their `IS_EXTERNAL` property set to `false` (with `_` referencing the individual nodes, and `isExternal` a property directive which accesses their `IS_EXTERNAL` property), followed by  a property directive `name` which returns the values of the `NAME` property of the nodes that passed the _Filter Step_, and finally an _Execution Directive_ `toList` which executes the traversal and returns the results in a list.

A shorter version of a query which returns the same results as the one above can be written using a _Property-Filter Step_. Property-filter steps are _Filter Steps_ which continue the traversal only for nodes which have a specific value in the property the _Property Filter Step_ refers to:

```java
joern> cpg.method.isExternal(false).name.toList 
res11: List[String] = List("main")
```

Dissecting the query again, `cpg` is the root object, `method` is a node-type step, `isExternal(false)` is a property-filter step that filters for nodes which have `false` as the value of their `IS_EXTERNAL` property, `name` is a property directive, and `toList` is the execution directive you are already familiar with.

{{% notice tip %}}
Be careful not to mix up property directives with property-filter steps, they look awfully similar.
Consider that:

a) `cpg.method.isExternal(true).name.toList` returns all `METHOD` nodes which have the `IS_EXTERNAL` property set to `true` (in this case, 10 results)

b) `cpg.method.isExternal.toList` returns the value of the `IS_EXTERNAL` property for all `METHOD` nodes in the graph (12 results)

c) `cpg.method.isExternal.name.toList` is an invalid query which will not execute
{{% /notice %}}

A final _Filter Step_ we will look at is named `where`. Unlike `filter`, this doesn't take a simple predicate `A => Boolean`, but instead takes a `Traversal[A] => Traversal[_]`. I.e. you supply a traversal which will be executed at the current position. The resulting Traversal will preserves elements if the provided traversal has _at least one_ result. The previous query that used a _Property Filter Step_ can be re-written using `where` like so:

```java
joern> cpg.method.where(_.isExternal(false)).name.toList 
res24: List[String] = List("main")
```

Maybe not particularly useful-seeming given this specific example, but keep it in the back of your head, because `filter` is a handy tool to have in the toolbox. Next up, _Map Steps_.

### Map Steps

_Map Steps_ are traversals that map a set of nodes into a different form given a function. _Map Steps_ are a powerful mechanism when you need to transform results to fit your specifics. For example, say you'd like to return both the `IS_EXTERNAL` and the `NAME` properties of all `METHOD` nodes in `X42`'s Code Property Graph. You can achieve that with the following query:

```java
joern> cpg.method.map(node => (node.isExternal, node.name)).toList
res6: List[(Boolean, String)] = List(
  (false, "main"),
  (true, "fprintf"),
  (true, "exit"),
  (true, "<operator>.logicalAnd"),
  (true, "<operator>.equals"),
  (true, "<operator>.greaterThan"),
  (true, "strcmp"),
  (true, "<operator>.indirectIndexAccess"),
  (true, "printf")
)
```

Don't be intimidated by the syntax used in the `map` _Step_ above. If you examine `map(node => (node.isExternal, node.name))` for a bit, you might be able to infer that the first `node` simply defines the variable that represents the node which preceeds the `map` _Step_, that the ASCII arrow `=>` is just syntax that preceeds the body of a lambda function, and that `(node.isExternal, node.name)` means that the return value of the lambda is a list which contains the value of the `isExternal` and `name` _Property Directives_ for each of the nodes matched in the previous step and also passed into the lambda. In most cases in which you need `map`, you can simply follow the pattern above. But should you ever feel constrained by the common pattern shown, remember that the function for the `map` step is written in the Scala programming language, a fact which opens up a wide range of possibilities if you invest a little time learning the language.

### Side Effect Steps

_Side Effect Steps_ are traversal steps that perform an action or modify the state of the traversal without altering the path of the traversal itself. They do not directly contribute to the results that are returned, but they might be used to store information, log data, or manipulate variables during traversal. These steps can be thought of as adding "side effects" to the traversal that can be useful for various purposes like counting, aggregating, or modifying data.

### Terminal Steps

_Terminal Steps_ are steps that end the traversal and return the final result. Once a terminal step is reached, the traversal is considered complete, and it provides the output in some form (e.g., a list, a set, or a single element). Unlike intermediate steps that continue building the traversal, terminal steps execute the traversal and stop further processing. After a terminal step, the traversal cannot be continued or extended; it’s finished.

## Traversal Steps

The steps described below are available when called on an `Iterator`. For these to be available, the following packaged must be imported, i.e., `import flatgraph.traversal.language.*`.

#### Basic steps
Assuming you have an `Iterator[X]`, where `X` is typically a domain specific type, but could also be flatgraph's root type for nodes [`GNode`](https://github.com/joernio/flatgraph/blob/92f4cc4b84bf6b8315971128995a75872376dcff/core/src/main/java/flatgraph/GNode.java), here's a (non-exhaustive) list of basic traversal steps.

| Name                    | Type        | Notes                                                                                                       |
| ----------------------- | ----------- | ----------------------------------------------------------------------------------------------------------- |
| **and**                 | Filter      | Only preserves elements for which _all of_ the given traversals have at least one result.                   |
| **cast[B]**             | Map         | Casts all elements to given type `B`.                                                                       |
| **choose**              | Filter      | Allows to implement conditional semantics: if, if/else, if/elseif, if/elseif/else, ...                      |
| **coalesce**            | Filter      | Evaluates the provided traversals in order and returns the first traversal that emits at least one element. |
| **collectAll[B]**       | Filter      | Collects all elements of the provided class `B` (beware of type-erasure).                                   |
| **dedup**               | Filter      | Deduplicate elements of this traversal - a.k.a. distinct, unique.                                           |
| **dedupBy**             | Filter      | Deduplicate elements of this traversal by a given function.                                                 |
| **discardPathTracking** | Side Effect | Disables path tracking, and any tracked paths so far.                                                       |
| **enablePathTracking**  | Side Effect | Enable path tracking - prerequisite for path/simplePath steps.                                              |
| **filter**              | Filter      | Filters in everything that evaluates to _true_ by the given transformation function.                        |
| **filterNot**           | Filter      | Filters in everything that evaluates to _false_ by the given transformation function.                       |
| **groupCount**          | Map         | Group elements and count how often they appear.                                                             |
| **groupCount[B]**       | Map         | Group elements by a given transformation function and count how often the results appear.                   |
| **head**                | Terminal    | The first element of the traversal.                                                                         |
| **is**                  | Filter      | Filters in everything that _is_ the given value.                                                            |
| **or**                  | Filter      | Only preserves elements for which _at least one of_ the given traversals has at least one result.           |
| **l/toSet/toSeq**       | Terminal    | Execute the traversal and returns the result as a list, set, or indexed sequence respectively.              |
| **last**                | Terminal    | The last element of the traversal.                                                                          |
| **not**                 | Filter      | Filters out everything that _is not_ the given value. Alias for `whereNot`.                                 |
| **path**                | Terminal    | Retrieve entire paths that have been traversed thus far.                                                    |
| **repeat**              | Map         | Repeat the given traversal.                                                                                 |
| **sideEffect**          | Side Effect | Perform side effect without changing the contents of the traversal.                                         |
| **simplePath**          | Filter      | Ensure the traversal does not include any paths that visit the same node more than once.                    |
| **size**                | Terminal    | Total size of elements in the traversal.                                                                    |
| **sorted**              | Map         | Sort elements by their natural order.                                                                       |
| **sortBy**              | Map         | Sort elements by the value of the given transformation function.                                            |
| **union**               | Filter      | Union/sum/aggregate/join given traversals from the current point.                                           |
| **within**              | Filter      | Filters out all elements that are _not_ in the provided set.                                                |
| **without**             | Filter      | Filters out all elements that _are_ in the provided set.                                                    |
| **where**               | Filter      | Only preserves elements if the provided traversal has at least one result.                                  |
| **whereNot**            | Filter      | Only preserves elements if the provided traversal does _not_ have any results.                              |

#### Node Steps

When starting the traversal from an `Iterator` of nodes [`GNode`](https://github.com/joernio/flatgraph/blob/92f4cc4b84bf6b8315971128995a75872376dcff/core/src/main/java/flatgraph/GNode.java).

| Name              | Type       | Notes                                                                                    |
| ----------------- | ---------- | ---------------------------------------------------------------------------------------- |
| **both**          | Map/Filter | Follow both in and out-neighbours for a given node. Can be restricted by edge type.      |
| **bothE**         | Map/Filter | Follow both in and out-edges for a given node. Can be restricted by edge type.           |
| **hasLabel**      | Filter     | Filters in nodes that match the given labels. Alias for `label`                          |
| **id**            | Map/Filter | Return a unique identifier(s) for the node(s) in the traversal. Can filter by given IDs. |
| **in**            | Map/Filter | In-neighbours for a given node. Can be restricted by edge type.                          |
| **inE**           | Map/Filter | In-edges for a given node. Can be restricted by edge type.                               |
| **out**           | Map/Filter | Out-neighbours for a given node. Can be restricted by edge type.                         |
| **outE**          | Map/Filter | Out-edges for a given node. Can be restricted by edge type.                              |
| **property**      | Map        | Retrieve the value for a single property for the defined property name.                  |
| **propertiesMap** | Map        | Retrieves all entity properties as a map.                                                |
| **label**         | Map/Filter | Node label. Can filter by given labels.                                                  |
| **labelNot**      | Filter     | Inverse of `label`.                                                                      |

#### Edge Steps

When starting the traversal from an `Iterator` of nodes [`Edge`](https://github.com/joernio/flatgraph/blob/92f4cc4b84bf6b8315971128995a75872376dcff/core/src/main/scala/flatgraph/Edge.scala).

| Name    | Type | Notes                                             |
| ------- | ---- | ------------------------------------------------- |
| **src** | Map  | Traverse to the source node (out-going node).     |
| **dst** | Map  | Traverse to the destination node (incoming node). |

## Property Directives

The steps described below are available when called on the entity/object directly. These are available as methods or properties on the objects so no import is necessary.

#### Graph Steps

Steps available from an instance of [`Graph`](https://github.com/joernio/flatgraph/blob/92f4cc4b84bf6b8315971128995a75872376dcff/core/src/main/scala/flatgraph/Graph.scala).

| Name          | Type     | Notes                                                                              |
| ------------- | -------- | ---------------------------------------------------------------------------------- |
| **allNodes**  | Map      | The nodes of the graph.                                                            |
| **allEdges**  | Map      | The edges of the graph.                                                            |
| **edgeCount** | Terminal | The total edges in the graph. Can be restricted by a given label.                  |
| **nodes**     | Filter   | Create a traversal from the nodes of the graph that match the given IDs or labels. |
| **nodeCount** | Terminal | Total nodes in the graph. Can be restricted by a given label.                      |

#### Edge Steps

Steps available from an instance of [`Edge`](https://github.com/joernio/flatgraph/blob/92f4cc4b84bf6b8315971128995a75872376dcff/core/src/main/scala/flatgraph/Edge.scala).

| Name             | Type | Notes                                          |
| ---------------- | ---- | ---------------------------------------------- |
| **label**        | Map  | The edge label.                                |
| **propertyName** | Map  | The property value of the edge, if one exists. |

#### Node Steps

Steps available from an instance of [`GNode`](https://github.com/joernio/flatgraph/blob/92f4cc4b84bf6b8315971128995a75872376dcff/core/src/main/java/flatgraph/GNode.java).

| Name      | Type | Notes                                                |
| --------- | ---- | ---------------------------------------------------- |
| **graph** | Map  | The graph this node belongs to.                      |
| **id**    | Map  | The node identifier.                                 |
| **label** | Map  | The node label.                                      |
