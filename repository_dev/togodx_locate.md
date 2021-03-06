# TogoDX locate SPARQList (Parameter 変更) 22.01.18

- 元 map_ids_to_attribute
- 入れ子 SPARQList の parameters もそのうち修正する
  - categoryIds -> nodes
  - togokey -> dataset
  - queryIds -> queries
- DAVI p-value (EASE Score)
  - https://david.ncifcrf.gov/content.jsp?file=functional_annotation.html

## Parameters

* `attribute`
  * default: gene_high_level_expression_refex
* `node`
  * example:
* `dataset`
  * default: uniprot
* `queries` (IDs from "Map your IDs")
  * default: ["Q9NYF8","Q4V339","A6NCE7","A7E2F4","P69849","A6NN73","Q92928","Q5T1J5","P0C7P4","Q6DN03","P09874","Q08211","Q5T4S7","P12270","Q9UPN3","P07814","P53621","P49321","P0C629","Q9BZK8","Q9BY65"]
  
## `pValueFlag`
```javascript
async ({attribute})=>{
  const fetchReq = async (url, options, body) => {
    console.log(url + " " + body);  // debug
    if (body) options.body = body;
    return await fetch(url, options).then(res=>res.json());
  }

  const togoDxAttributes = "https://raw.githubusercontent.com/togodx/togodx-config-human/develop/config/attributes.json";
  const attributesJson = await fetchReq(togoDxAttributes, {method: "get"});
  const attributeDataset = attributesJson.attributes[attribute].dataset;
  let obj = {attribute: attributesJson.attributes[attribute]};
  if (attributeDataset == "uniprot") {  // || attributeDataset == "pdb") {
    obj[attributeDataset] = true;
  } else if (attribute.match(/gene_biotype_ensembl$/)) {
    obj.ensembl_gene_biotype = true;  
  } else if (attribute.match(/gene_chromosome_ensembl$/)) {
    obj.ensembl_gene_chromosome = true;  
  } else if (attribute.match(/gene_number_of_exons_ensembl$/)) {
    obj.ensembl_transcript_exons = true;  
  } else if (attribute.match(/gene_number_of_paralogs_homologene$/)
            || attribute.match(/gene_evolutionary_conservation_homologene$/)) {
    obj.ncbigene_homologene = true;  
  } else if (attribute.match(/gene_\w+_level_expression_refex$/)) {
    obj.ncbigene_refex = true;  
  } else if (attribute.match(/gene_high_level_expression_gtex6$/)) {
    obj.ensembl_gene_gtex6 = true;  
  } else if (attribute.match(/gene_transcription_factors_chip_atlas$/)) {
    obj.ensembl_gene_chip_atlas = true;  
  }
  if (Object.keys(obj).length) return obj;
  obj.nonPValue = true;
  return obj;
}
```

## Endpoint
https://integbio.jp/togosite/sparql

## `population`
```sparql
PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX taxid: <http://identifiers.org/taxonomy/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX orth: <http://purl.org/net/orth#>
PREFIX refexo: <http://purl.jp/bio/01/refexo#>
PREFIX uniprot: <http://purl.uniprot.org/core/>
PREFIX taxon_up: <http://purl.uniprot.org/taxonomy/>
PREFIX pdbo: <https://rdf.wwpdb.org/schema/pdbx-v50.owl#>
{{#if pValueFlag.ensembl_gene_biotype}}
SELECT (COUNT(DISTINCT ?ensg) AS ?total_count)
FROM <http://rdf.integbio.jp/dataset/togosite/ensembl>
WHERE {
  ?ensg obo:RO_0002162 taxid:9606 ; # in taxon
      a ?type .
  ?enst obo:SO_transcribed_from ?ensg .
  FILTER regex(str(?type), "/terms/ensembl/")
}
{{/if}}
{{#if pValueFlag.ensembl_gene_chromosome}}
SELECT (COUNT(DISTINCT ?ensg) AS ?total_count)
FROM <http://rdf.integbio.jp/dataset/togosite/ensembl>
WHERE {
  ?enst obo:SO_transcribed_from ?ensg .
  ?ensg obo:RO_0002162 taxid:9606 ; # in taxon
    faldo:location ?ensg_location .
  BIND (strbefore(strafter(str(?ensg_location), "GRCh38/"), ":") AS ?chromosome)
  VALUES ?chromosome {
    "1" "2" "3" "4" "5" "6" "7" "8" "9" "10"
    "11" "12" "13" "14" "15" "16" "17" "18" "19" "20" "21" "22"
    "X" "Y" "MT"
  }
}
{{/if}}
{{#if pValueFlag.ensembl_transcript_exons}}
SELECT (COUNT(DISTINCT ?enst) AS ?total_count)
FROM <http://rdf.integbio.jp/dataset/togosite/ensembl>
WHERE {
  ?enst obo:SO_has_part ?exon ;
      obo:SO_transcribed_from ?ensg .
  ?ensg obo:RO_0002162 taxid:9606 . # in taxon
}
{{/if}}
{{#if pValueFlag.ncbigene_homologene}}
SELECT (COUNT(DISTINCT ?human_gene) AS ?total_count)
FROM <http://rdf.integbio.jp/dataset/togosite/homologene/data>
WHERE {
  ?human_gene orth:taxon taxid:9606 .
}
{{/if}}
{{#if pValueFlag.ncbigene_refex}}
SELECT (COUNT(?gene) AS ?total_count)
FROM <http://rdf.integbio.jp/dataset/togosite/refex_id_relation_human>
WHERE {
  ?gene refexo:affyProbeset ?affy .
}
{{/if}}
{{#if pValueFlag.ensembl_gene_gtex6}}
SELECT (COUNT(DISTINCT ?ensg) AS ?total_count)
FROM <http://rdf.integbio.jp/dataset/togosite/refex_tissue_specific_genes_gtex_v6>
WHERE {
  ?ensg a refexo:GTEx_v6_ts_evaluated_gene
}
{{/if}}
{{#if pValueFlag.ensembl_gene_chip_atlas}}
SELECT (COUNT(?ensg) AS ?total_count)
FROM <http://rdf.integbio.jp/dataset/togosite/chip_atlas_ncbigene_ensembl>
WHERE {
  ?ensg refexo:ncbigene ?ncbigene .
}
{{/if}}
{{#if pValueFlag.uniprot}}
SELECT (COUNT (DISTINCT ?uniprot) AS ?total_count)
FROM <http://rdf.integbio.jp/dataset/togosite/uniprot>
WHERE {
  ?uniprot a uniprot:Protein ;
         uniprot:organism taxon_up:9606 ;
         uniprot:proteome ?proteome .
  FILTER(REGEX(STR(?proteome), "UP000005640"))
}
{{/if}}
{{#if pValueFlag.pdb}}
SELECT (COUNT(DISTINCT ?pdb) AS ?total_count)
FROM <http://rdf.integbio.jp/dataset/togosite/pdbj>
WHERE {
  ?pdb a pdbo:datablock ;
      pdbo:has_pdbx_entity_nonpolyCategory ?nonpoly ;
      pdbo:has_entityCategory/pdbo:has_entity/rdfs:seeAlso taxid:9606 .
}
{{/if}}
{{#if pValueFlag.nonPValue}}
SELECT ?s
WHERE {
}
{{/if}}  
```

## `distribution`
```javascript
async ({node, queries, dataset, pValueFlag, population})=>{
  const fetchReq = async (url, body) => {
    let options = {	
      method: 'POST',
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/x-www-form-urlencoded'
      }
    }
    if (body) options.body = body;
    console.log(url);
    console.log(body);
    return await fetch(url, options).then(res=>res.json());
  }

  const attributeDataset = pValueFlag.attribute.dataset;
  const api = pValueFlag.attribute.api;
  const togoidSparqlistSplitter = "togoid_sparqlist_splitter"; // nested SPARQLet relative path
  const sparqlistSplitter = "sparqlist_splitter"; // nested SPARQLet relative path
  const togoidApi = "togoid_route_sparql"; // nested SPARQLet relative path
  let idLimit = 2000; // split 判定
  if (attributeDataset == "chembl_compound") idLimit = 500; // restrict POST response size
  const sparqlet = api.split(/\//).slice(-1)[0];  // nested SPARQLet relative path

  // convert user IDs to primary IDs for SPARQLet
  queries = JSON.parse(queries).join(","); // 歴史的経緯の変換
  let converted_queries = "";
  if (dataset != attributeDataset) {
    let body = "source=" + dataset + "&target=" + attributeDataset + "&ids=" +  encodeURIComponent(queries);
    let togoidPair;
    if (queries.split(/,/).length <= idLimit) {
      togoidPair = await fetchReq(togoidApi, body);
    } else {
      body += "&sparqlet=" + encodeURIComponent(togoidApi) + "&limit=" + idLimit;
      togoidPair = await fetchReq(togoidSparqlistSplitter, body).map(d=>d.target_id).join(",");
    }
    converted_queries = togoidPair.map(d=>d.target_id).join(",");
  } else {
    converted_queries = queries;
  }
  
  if (!converted_queries.match(/\w/)) return [];
  
  // get property data
  let distribution = [];
  let body = "queryIds=" + converted_queries;  // #### 入れ子 SPARQList. 要パラメータ名の整理
  if (node) body += "&categoryIds=" + node;  // #### 入れ子 SPARQList. 要パラメータ名の整理
  if (converted_queries.split(/,/).length <= idLimit) distribution = await fetchReq(sparqlet, body);
  body += "&sparqlet=" + encodeURIComponent(sparqlet) + "&limit=" + idLimit;
  distribution = await fetchReq(sparqlistSplitter, body);
  let hit = {};
  for (let d of distribution) {
    hit[d.categoryId] = Number(d.count);
  }
  
  // get unfiltered data
  body = false;
  if (node) body = "categoryIds=" + node;  // #### 入れ子 SPARQList. 要パラメータ名の整理
  let originalDistribution = await fetchReq(sparqlet, body);
  for (let i = 0; i < originalDistribution.length; i++) {
    let hit_tmp = 0;
    if (originalDistribution[i].categoryId) {
      originalDistribution[i].node = originalDistribution[i].categoryId;
      delete(originalDistribution[i].categoryId);
    }
    if (hit[originalDistribution[i].node]) hit_tmp = hit[originalDistribution[i].node];
    originalDistribution[i].mapped = hit_tmp;
  }

  // without p-value
  if (pValueFlag.nonPValue) return originalDistribution;

  // with pvalue (gene, protein)
  let calcPvalue = (a, b, c, d) => {
    // 不正数値検出
    let maxLimit = 300000; // ensembl_transcript: ~253,000
    // console.log([a,b,c,d].join(","));
    if (a < 0 || b < 0 || c < 0 || d < 0) return false;
    if (a > maxLimit || b > maxLimit || c > maxLimit || d > maxLimit) return false;
    
    let sigDigi = (num, exp) => {
      while (num > 10) {
        num /= 10;
        exp++;
      }
      while (num < 1) {
        num *= 10;
        exp--;
      }
      return [num, exp];
    }
    
    let calcProb = (a, b, c, d) => {
      // prob = num * 10 ** exp;
      let num = 1;
      let exp = 0; 
      for (let i = 1;         i <= a;             i++) { [num, exp] = sigDigi(num / i, exp); }  // 1/a!
      for (let i = b + 1;     i <= a + b;         i++) { [num, exp] = sigDigi(num * i, exp); }  // (a+b)!/b!
      for (let i = c + 1;     i <= a + c;         i++) { [num, exp] = sigDigi(num * i, exp); }  // (a+c)!/c!
      for (let i = d + 1;     i <= c + d;         i++) { [num, exp] = sigDigi(num * i, exp); }  // (c+d)!/d!
      for (let i = b + d + 1; i <= a + b + c + d; i++) { [num, exp] = sigDigi(num / i, exp); }  // (b+d)!/(a+b+c+d)!
      return num * 10 ** exp;
    }
    
    let cutoffProb = calcProb(a, b, c, d);
  
    let max = a + b;
    if (max > a + c) max = a + c;
 
    let pValue = 0;
    for (let i = 0; i <= max; i++) {
      let delta = a - i;
      let tmpProb = 1;
      if (b + delta >= 0 && c + delta >= 0 && d - delta >= 0) tmpProb = calcProb(i, b + delta, c + delta, d - delta);
      if (tmpProb <= cutoffProb) pValue += tmpProb;
    }
    if (pValue > 0.9999) pValue = 1; // 有効数字このくらい？
    return pValue;
  }
  
  const PT = Number(population.results.bindings[0].total_count.value);
  const LT = converted_queries.split(/,/).length;
  
  return originalDistribution.map(d => {
    const LH = d.mapped;
    const PH = d.count;
    if (LH == 0) return d;
    if (LH == 1) d.pvalue = 1;
    else d.pvalue = calcPvalue(
      LH - 1, 
      LT - LH, 
      PH - LH + 1, 
      PT - LT - (PH - LH)
    );
    return d;
  })
}
```