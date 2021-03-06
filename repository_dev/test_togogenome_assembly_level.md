# test_togogenome_assembly_level (信定)

## Endpoint
http://togogenome.org/sparql

## `main`
```sparql
PREFIX asm: <http://ddbj.nig.ac.jp/ontologies/assembly/>
PREFIX taxid:<http://identifiers.org/taxonomy/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
SELECT DISTINCT  ?GCFnumber  ?asm_name  ?assembly_level
FROM <http://togogenome.org/graph/assembly_report>
WHERE
{
  ?assembly_report a asm:Assembly_Database_Entry ;
                   rdfs:seeAlso ?GCF ;
                   asm:asm_name ?asm_name ;
                   asm:assembly_level ?assembly_level .
  BIND (strafter(str(?GCF), "http://ddbj.nig.ac.jp/ontologies/assembly/") AS ?GCFnumber)
}
```
## `return`

```javascript
({main}) => {
  
  let tree = [
    {
      id: "root",
      label: "root node",
      root: true
    }
  ];

  let edge = {};
  main.results.bindings.map(d => {
    tree.push({
      id: d.GCFnumber.value,
      label: d.asm_name.value,
      leaf: true,
      parent: d.assembly_level.value
    })
  // root との親子関係を追加
    if (!edge[d.assembly_level.value]) {
      edge[d.assembly_level.value] = true;
      tree.push({   
        id: d.assembly_level.value,
        label: d.assembly_level.value,
        leaf: false,
        parent: "root"
      })
    }
  });
  
  return tree;
};
```


