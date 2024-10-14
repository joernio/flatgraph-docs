+++
title = "Traversal steps"
weight = 3
+++

The most important basic traversal steps will be the ones generated for your domain as highlighted in [glimpse-of-a-simple-use-case](index.html#glimpse-of-a-simple-use-case).

In addition to the generated domain-specific steps based on your schema, there's some basic traversal steps that you can use generically across domains, e.g. to traverse from a node (or an `Iterator[Node]`) to their neighbors, lookup their properties etc. 
There are also more advanced steps like `repeat` and advanced features like path tracking which will be described further below. 

{{% notice tip %}}
flatgraph traversals are based on Scala's Iterator, so you can also use all regular [collection methods](https://docs.scala-lang.org/scala3/book/collections-methods.html). 
{{% /notice %}}


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


#### Graph Steps

When starting the traversal from the graph object, i.e., an instance of [`Graph`](https://github.com/joernio/flatgraph/blob/92f4cc4b84bf6b8315971128995a75872376dcff/core/src/main/scala/flatgraph/Graph.scala).

| Name          | Type | Notes |
| ------------- | ---- | ----- |
| **edgeCount** | Map  | Graph | The total edges in the graph. |
| **nodeCount** | Map  | Graph | Total nodes in the graph.     |


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

