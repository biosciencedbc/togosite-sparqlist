# TogoGenome から各生物種のゲノムサイズ （信定）

## Endpoint
http://togogenome.org/sparql

## `genome_size`
```sparql
PREFIX taxid: <http://identifiers.org/taxonomy/>
PREFIX stats: <http://togogenome.org/stats/>
SELECT distinct ?tax ?tax_id ?genome_size 
FROM <http://togogenome.org/graph/stats>
where{
  ?tax stats:sequence_length ?genome_size .
  FILTER contains (str(?tax), "taxonomy")
  BIND (strafter(str(?tax), "http://identifiers.org/taxonomy/") AS ?tax_id)
}
limit 10
```

## `query`
```javascript
({genome_size}) => {
let tree = [] ;
genome_size.results.bindings.map(d => {
  tree.push ( d.tax.value ) ;
}) ;
return tree ;
}
```

## `organism`
```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX dcterm: <http://purl.org/dc/terms/>
SELECT distinct ?tax ?tax_id  ?label
FROM <http://togogenome.org/graph/taxonomy>
WHERE {

  values ?tax { {{#each query}} <{{this}}> {{/each}} }

?tax a <http://ddbj.nig.ac.jp/ontologies/taxonomy/Taxon> ; 
  rdfs:label ?label ;
  dcterm:identifier ?tax_id .
  }
limit 10
```
## `return`
```javascript
({genome_size, organism}) => {
  
  let tree = [
    {
      id: "root",
      label: "root node",
      root: true
    }
  ];

  let edge = {};
  genome_size.results.bindings.map(d => {
    tree.push({
      id: d.tax_id.value,
      label: d.genome_size.value,
      leaf: true,
      parent: d.tax_id.value
    });
    });
  let edge2 = {};
  organism.results.bindings.map(d => {
    tree.push({
      id: d.tax_id.value,
      label: d.label.value,
      leaf: false,
      parent: "root"
    });
    });
  
  return tree;
}
```


