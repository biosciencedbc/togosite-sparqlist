# Transcript attributes from Ensembl RDF（井手・八塚・池田）

ENSG ID を受け取って、子の transcript の情報を返す

## Endpoint

https://integbio.jp/rdf/ebi/sparql

## Parameters
* `ensg`
  * default: ENSG00000171097
  * example: ENSG00000171097

## `main`

```sparql
PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX taxon: <http://identifiers.org/taxonomy/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX biohack: <http://biohackathon.org/resource/faldo#>
PREFIX enst: <http://rdf.ebi.ac.uk/resource/ensembl.transcript/>
PREFIX ensg: <http://rdf.ebi.ac.uk/resource/ensembl/>
PREFIX dc: <http://purl.org/dc/elements/1.1/>
PREFIX ensembl: <http://identifiers.org/ensembl/>

SELECT ?enst_id ?gene_name ?label ?location ?type_name (GROUP_CONCAT(DISTINCT ?uniprot_id; separator=",") AS ?uniprot_ids) (COUNT(DISTINCT ?exon) AS ?exon_count)
FROM <http://rdf.ebi.ac.uk/dataset/ensembl/102/homo_sapiens>
WHERE
{
  VALUES ?ensg { ensg:{{ensg}} }
  ?enst obo:SO_transcribed_from ?ensg .
  ?ensg dc:description ?gene_name .
  ?enst a ?type ;
        rdfs:label ?label ;
        dc:identifier ?enst_id ;
        biohack:location/rdfs:label ?location ;
        obo:SO_has_part ?exon .
  OPTIONAL {
    ?enst obo:SO_translates_to ?ensp .
    ?ensp rdfs:seeAlso ?uniprot .
    FILTER(CONTAINS(STR(?uniprot), "http://identifiers.org/uniprot/"))
    BIND(STRAFTER(STR(?uniprot), "http://identifiers.org/uniprot/") AS ?uniprot_id)
  }
  FILTER REGEX(?type, "^http://rdf")
  BIND(REPLACE(STR(?type), "http://rdf.ebi.ac.uk/terms/ensembl/", "") AS ?type_name)
}ORDER BY ?enst_id
```

## `return`

```javascript
({ main }) => {
  return main.results.bindings.map((elem) => ({
    enst_id: elem.enst_id.value,
    enst_url: "http://identifiers.org/ensembl/" + elem.enst_id.value,
    uniprot_id: elem.uniprot_ids.value.split(",").map((elem)=>("<a href=\"http://identifiers.org/uniprot/"+elem+"\">"+elem+"</a>")).join(", "),
    //uniprot_url: "http://identifiers.org/uniprot/" + elem.uniprot_id?.value,
    type: elem.type_name.value.replace(/_/g, " "),
    label:  elem.label.value,
    location: elem.location.value,
    exon_count: elem.exon_count.value
  }))
}
```