+++
title = "Traversal steps"
weight = 2
+++

The most important basic traversal steps will be the ones generated for your domain as highlighted in [glimpse-of-a-simple-use-case](index.html#glimpse-of-a-simple-use-case).

In addition to the generated domain-specific steps based on your schema, there's some basic traversal steps that you can use generically across domains, e.g. to traverse from a node (or an `Iterator[Node]`) to their neighbors, lookup their properties etc. 
There are also more advanced steps like `repeat` and advanced features like path tracking which will be described further below. 

{{% notice tip %}}
flatgraph traversals are based on Scala's Iterator, so you can also use all regular [collection methods](https://docs.scala-lang.org/scala3/book/collections-methods.html). 
{{% /notice %}}


#### Basic steps

| Name                  | Default           | Notes       |
|-----------------------|-------------------|-------------|
| **page**              | _&lt;empty&gt;_   | Mandatory reference to the page. |
| **onempty**           | `disable`       | Defines what to do with the button if the content overlay is empty:<br><br>- `disable`: The button is displayed in disabled state.<br>- `hide`: The button is removed. |
| **onwidths**          | _&lt;varying&gt;_ | The action, that should be executed if the site is displayed in the given width:<br><br>- `show`: The button is displayed in its given area<br>- `hide`: The button is removed.<br>- `area-XXX`: The button is moved from its given area into the area `XXX`. |
| **onwidthm**          | _&lt;varying&gt;_ | See above. |
| **onwidthl**          | _&lt;varying&gt;_ | See above. |
