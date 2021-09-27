## Endpoint

https://integbio.jp/togosite/sparql

## Parameters

* `id`
  * default: ENSG00000150773
  * example: ENSG00000150773

## `main`

```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX idt_ensg: <http://identifiers.org/ensembl/>
PREFIX dc: <http://purl.org/dc/elements/1.1/>
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX taxon: <http://identifiers.org/taxonomy/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX refexo: <http://purl.jp/bio/01/refexo#>
PREFIX schema: <http://schema.org/>
PREFIX ensg: <http://rdf.ebi.ac.uk/resource/ensembl/>

SELECT DISTINCT ?ensg_id ?idt_ensg ?gene_symbol ?desc ?type_label ?location
  (GROUP_CONCAT(DISTINCT ?gtex_tissue_label; separator=", ") AS ?gtex_tissue_labels)
  (GROUP_CONCAT(DISTINCT ?hpa_tissue_label; separator=", ") AS ?hpa_tissue_labels)
  (GROUP_CONCAT(DISTINCT ?hpa_cell_label; separator=", ") AS ?hpa_cell_labels)
WHERE {
  VALUES ?ensg { ensg:{{id}} }
  VALUES ?idt_ensg { idt_ensg:{{id}} }
  GRAPH <http://rdf.integbio.jp/dataset/togosite/ensembl> {
    ?ensg obo:RO_0002162 taxon:9606 ;
             dc:identifier ?ensg_id ;
             rdfs:label ?gene_symbol ;
             dc:description ?desc ;
             faldo:location ?loc ;
             a ?type .
    FILTER(STRSTARTS(STR(?type), "http://rdf.ebi.ac.uk/terms/ensembl/"))
    BIND(REPLACE(STRAFTER(STR(?type), "http://rdf.ebi.ac.uk/terms/ensembl/"), "_", " ") as ?type_label)
    ?loc rdfs:label ?location .
  }
  OPTIONAL {
    GRAPH <http://rdf.integbio.jp/dataset/togosite/refex_tissue_specific_genes_gtex_v6> {
      ?idt_ensg refexo:isPositivelySpecificTo ?gtex_tissue .
    }
    GRAPH <http://rdf.integbio.jp/dataset/togosite/refexsample_gtex_v8_summary> {
      VALUES ?name {"cell type" "tissue"}
      ?refexs schema:additionalProperty [
        schema:name ?name ;
        schema:value ?gtex_tissue_label ;
        schema:valueReference ?gtex_tissue
      ] .
    }
  }
  OPTIONAL {
    GRAPH <http://rdf.integbio.jp/dataset/togosite/hpa_cell_specificity> {
      ?idt_ensg refexo:isPositivelySpecificTo ?hpa_cell .
    }
    GRAPH <http://rdf.integbio.jp/dataset/togosite/caloha> {
      ?hpa_cell rdfs:label ?hpa_cell_label .
    }
  }
  OPTIONAL {
    GRAPH <http://rdf.integbio.jp/dataset/togosite/hpa_tissue_specificity> {
      ?idt_ensg refexo:isPositivelySpecificTo ?hpa_tissue .
    }
    GRAPH <http://rdf.integbio.jp/dataset/togosite/caloha> {
      ?hpa_tissue rdfs:label ?hpa_tissue_label .
    }
  }
}
```

## `return`

```javascript
({ main, id }) => {
  let binding = main.results.bindings[0];
  let objs = [{
    "Ensembl ID": binding.ensg_id.value,
    "Ensembl URL": binding.idt_ensg.value,
    "Gene symbol": binding.gene_symbol.value,
    "Description": binding.desc.value,
    "Gene type": binding.type_label.value,
    "Location": binding.location.value,
    "Tissue specificity (GTEx)": binding.gtex_tissue_labels.value,
    "Tissue specificity (HPA)": binding.hpa_tissue_labels.value,
    "Cell specificity (HPA)": binding.hpa_cell_labels.value,
    "Expression": "<a href=\"https://gtexportal.org/home/gene/" + id + "\">" + "View Expression at GTEx Portal</a>, " +
                  "<a href=\"https://www.proteinatlas.org/" + id + "\">" + "View Expression at ProteinAtlas</a>"
  }];

  if (objs[0]["Tissue specificity (GTEx)"] == "") {
    objs[0]["Tissue specificity (GTEx)"] = "(Low tissue specificity)";
  }
  if (objs[0]["Tissue specificity (HPA)"] == "") {
    objs[0]["Tissue specificity (HPA)"] = "(Low tissue specificity)";
  }
  if (objs[0]["Cell specificity (HPA)"] == "") {
    objs[0]["Cell specificity (HPA)"] = "(Low cell specificity)";
  }
  return objs;
};
```