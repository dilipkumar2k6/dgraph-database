# Reference 
https://www.youtube.com/watch?v=vhatoyZ2vic
# Mutation
- Addtion or removing data in DGraph is called mutation.
- A mutation that add triples is done with the `set` keyword
```
{
    set {
        # tripels in here
    }
}
```
# Triples
The input language is triples in the W3C standard RDF N-Quad format. 
Each triplet has following form
```
<subject> <predicate> <object> .
```
- It means, the graph node identified by `Subject` is linked to `Object` with directed edge `predicate`
- Each triplets ends with period `.`
- The subject of triplet is always node in the graph
- The object may be node or value (a literal)
Following is an example
```
<0x1> <name> "Alice" .
<0x1> <dgraph.type> "Person" . 
```
This represents a graph node with ID `0x01` has a name with string value "Alice"

```
<0x01> <friend> <0x02>
```
This represents a graph node `0x01` is liked with `friend` edge to node `0x02`

- DGraph creates a unique `64 bit` identifier for every blank node in the mutation - the node's UUID
- A mutation can include a blank node as an identifier for the subject or object
- Or a known UID from a previous mutation
# Blank Nodes and UID
- Blank nodes in mutation, written `_:identifier`, identify nodes within a mutation.
- Dgraph creates UID identifying each blank node and returns the created UID as mutation results
Following is example for blank nodes
```
{
    set {
        _:class <student> _:x .
        _:class <student> _:y .
        _:class <name> "awesome class" .
        _:class <dgraph.type> "Class" .
        _:x <name> "Alice" .
        _:x <dgraph.type> "Person" .
        _:x <dgraph.type> "Student" .
        _:x <planet> "Mars" .
        _:x <friend> _:y .
        _:y <name> "bob" .
        _:y <dgraph.type> "Person" .
        _:y <dgraph.type> "Class" .
    }
} 
```
Following is result for this mutation 
```
{
  "data": {
    "code": "Success",
    "message": "Done",
    "uids": {
      "class": "0x2712",
      "x": "0x2713",
      "y": "0x2714"
    }
  }
}
```