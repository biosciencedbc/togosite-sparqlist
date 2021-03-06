# ChIP-Atlas の転写因子を GO BP で分類(池田)

uniprot GO 共有 SPARQLet を流用

- 指定 GO の1階層下の内訳
  - すでに最下層で、下層が無い場合は空配列を返す

## Parameters

* `categoryIds` (type: go) (Req.)
  * default: GO_0003674
  * example: GO_0008150 (biological process), GO_0005575 (cellular component), GO_0003674 (molecular function), ... 
* `queryIds` (type: ENSG ID of TF)
  * example: ENSG00000065978, ENSG00000070495, ENSG00000072501, ENSG00000079246, ENSG00000099381, ENSG00000102974, ENSG00000104856
* `mode`
  * example: idList, objectList

## `queryArray`
- Query Ensembl Gene ID を配列に
```javascript
({queryIds}) => {
  queryIds = queryIds.replace(/,/g," ")
  if (queryIds.match(/[^\s]/)) return queryIds.split(/\s+/);
  return false;
}
```

## `categoryArray`
- UniProt keyword ID を配列に
```javascript
({categoryIds})=>{
  categoryIds = categoryIds.replace(/,/g, " ");
  if (categoryIds.match(/[^\s]/)) {
    let array = categoryIds.split(/\s+/);
    let categoryArray = [];
    for (let id of array) {
      if (!id.match(/^wo_GO_\d+/)) categoryArray.push(id);
      else categoryArray.push(id.match(/wo_(GO_\d+)/)[1]);
    }
    if (categoryArray.length == 0) return false;
    return categoryArray;
  } 
  return false;
}
```

## `withoutId`
```javascript
({categoryIds})=>{
  if (categoryIds == "GO_0008150" || categoryIds == "GO_0005575" || categoryIds == "GO_0003674" ) return categoryIds;
  if (categoryIds.match(/wo_GO_\d+/)) return categoryIds.match(/wo_(GO_\d+)/)[1];
  return false;
}
```

## `category_top_flag`
```javascript
({categoryIds})=>{
  if (categoryIds == "GO_0008150" || categoryIds == "GO_0005575" || categoryIds == "GO_0003674") return true;
  return false;
}
```

## Endpoint
https://integbio.jp/togosite/sparql

## `targetGo`
- 欲しい GO の階層を取る
  - UniProt エンドポイントのGO階層が展開されてしまってるため
  
```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX obo: <http://purl.obolibrary.org/obo/>
SELECT DISTINCT ?go (SAMPLE(?child_category) AS ?child)
FROM <http://rdf.integbio.jp/dataset/togosite/go>
WHERE
{
{{#if mode}}
  {{#if category_top_flag}}
  VALUES ?go_list { {{#each categoryArray}} obo:{{this}} {{/each}} }
  ?go rdfs:subClassOf ?go_list .
  {{else}}
  VALUES ?go { {{#each categoryArray}} obo:{{this}} {{/each}} }
  {{/if}}
{{else}}
  VALUES ?go_list { {{#each categoryArray}} obo:{{this}} {{/each}} }
  ?go rdfs:subClassOf ?go_list .
{{/if}}
  OPTIONAL {
    ?child_category rdfs:subClassOf ?go .
    # taxon:9606 ^up:organism/up:classifiedWith/rdfs:subClassOf ?child_category . # 重い
  }
}
```

## `targetGoArray`
- 単純な配列に

```javascript
({targetGo}) => {
  return targetGo.results.bindings.map(d => d.go.value.replace("http://purl.obolibrary.org/obo/", ""));
}
```

## `targetTf`
- ChIP-Atlas で扱っている転写因子の ENSG ID を取る
  - メインの SPARQL で一緒に取るほうが合理的だが、実用的な速さにならないのでここで予め取得しておく

```sparql
PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX ensg: <http://identifiers.org/ensembl/>

SELECT DISTINCT ?tf_ensg
FROM <http://rdf.integbio.jp/dataset/togosite/chip_atlas>
WHERE {
  {{#if queryArray}}
  VALUES ?tf_ensg { {{#each queryArray}} ensg:{{this}} {{/each}} }
  {{/if}}
  GRAPH <http://rdf.integbio.jp/dataset/togosite/chip_atlas> {
    ?tf_ensg obo:RO_0002428 ?target .
  }
}
```

## `targetTfArray`
- 単純な配列に

```javascript
({targetTf}) => {
  return targetTf.results.bindings.map(d => d.tf_ensg.value.replace("http://identifiers.org/ensembl/", ""));
}
```

## Endpoint
https://integbio.jp/togosite/sparql
 
## `withAnnotation`
- mode が指定されていないときのカウントは後で javascript 内でやる。それほど多くないので大丈夫

```sparql
PREFIX up: <http://purl.uniprot.org/core/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX ensg: <http://purl.uniprot.org/bgee/>

SELECT DISTINCT ?tf_ensg ?category ?label
FROM <http://rdf.integbio.jp/dataset/togosite/uniprot>
FROM <http://rdf.integbio.jp/dataset/togosite/go>
WHERE {
  VALUES ?tf_ensg { {{#each targetTfArray}} ensg:{{this}} {{/each}} }
  VALUES ?category { {{#each targetGoArray}} obo:{{this}} {{/each}} }
  ?uniprot a up:Protein ;
           up:classifiedWith/rdfs:subClassOf* ?category ;
           rdfs:seeAlso ?tf_ensg .
  ?category rdfs:label ?label .
}
```

- あるGOカテゴリを持たないUniProtを１つのSPARQLで取ろうとするとメモリオーバーするので変則的

## `withoutAnnotation`
- withAnnotation に出現しないが targetTfArray に出現するもの
```javascript
({mode, queryArray, targetTfArray, withAnnotation, withoutId}) => {
  if (!withoutId) return {results: {bindings: []}};
  let query = {};
  if (queryArray) {
    for (let d of queryArray) {
      query[d] = true;
    }
  }
  let withGo = {};
  for (let d of withAnnotation.results.bindings) {
    withGo[d.tf_ensg.value.replace("http://purl.uniprot.org/bgee/", "")] = true;
  }
  let bindings = [];
  for (let d of targetTfArray) {
    if (!withGo[d] && (!queryArray || (queryArray && query[d]))) {
      bindings.push({
        tf_ensg: {value: "http://purl.uniprot.org/bgee/" + d},
        category: {value: "wo_" + withoutId},
        label: {value: "without annotation"}
      });
    }
  }
  return {results: {bindings: bindings}};
}
```

## `return`
- 存在レベル、タンパク質リストでのフィルタリング
```javascript
({mode, category_top_flag, categoryArray, withoutId, withAnnotation, withoutAnnotation, targetGo}) => {
  const idVar = "tf_ensg";
  const idPrefix = "http://purl.uniprot.org/bgee/";
  const categoryPrefix = "http://purl.obolibrary.org/obo/";
  let data = [];
  if (categoryArray) data = withAnnotation.results.bindings;
  if (withoutId) {
    if (category_top_flag)
      data = data.concat(withoutAnnotation.results.bindings);
    else
      data = withoutAnnotation.results.bindings;
  }
  if (mode == "objectList") return data.map(d => {
    return {
      id: d[idVar].value.replace(idPrefix, ""), 
      attribute: {
        categoryId: d.category.value.replace(categoryPrefix, ""), 
        uri: d.category.value,
        label : d.label.value
      }
    }
  });
  if (mode == "idList") return data.map(d => d[idVar].value.replace(idPrefix, ""));

  let hasChild = {};
  for (let d of targetGo.results.bindings) {
    if (d.child) hasChild[d.go.value] = true;
  }
  let countData = [];
  let goInDataIndex = {};
  let n = 0;
  for (d of data) {
    if (!Number.isInteger(goInDataIndex[d.category.value])) {
      goInDataIndex[d.category.value] = n++;
      let obj = {
        categoryId: d.category.value.replace(categoryPrefix, ""), 
        label: d.label.value,
        count: 0
      }
      if (hasChild[d.category.value]) obj.hasChild = true;
      countData.push(obj);
    }
    countData[goInDataIndex[d.category.value]].count++;
  }
  return countData.sort((a, b) => {
    if (a.label === "without annotation" || b.label === "without annotation") return 1;
    if (a.count > b.count) return -1;
    return 1;
  });
}
```