# Reference 
https://www.youtube.com/watch?v=VM7METe3N3Q
# Query
- Every query has a name, and the result is labelled with the same name.
- The search criteria `func: ...` matches nodes. Function eq does what you’d expect, matching nodes with a name equalling “Michael”. The result is the matched nodes and listed outgoing edges from those nodes.
```
{
  find_michael(func: eq(name@., "Michael")) {
    uid
    name@.
    age
  }
}

```
# How Dgraph Search Works
- The graphs in Dgraph can be huge, so starting searching from all nodes isn’t efficient. Dgraph needs a place to start searching, that’s the root node.
- At root, we use `func:` and a function to find an initial set of nodes. So far we’ve used `eq` and `allofterms`for string search, but we can also search on other values like dates, numbers, and also filters on count.
- Dgraph needs to build an index on values that are to be searched in this way because, without an index to make the search efficient, Dgraph would have to crawl through the whole database to find matching values.
- From the set of nodes matched in the root filter, Dgraph then follows edges to satisfy the remainder of the query. The filters on blocks inside the root are only applied to the nodes reached by following listed edges to them.
- The root `func:` only accepts a single function and doesn’t accept AND, OR and NOT connectives as in filters. So the syntax query_name(func: foo(...)) @filter(... AND ...) {...} is required when further filtering on the elements returned by the root function is required.
- 






