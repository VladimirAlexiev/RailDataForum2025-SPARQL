# RailDataForum2025 SPARQL Tutorial

- Author: Vladimir Alexiev, chief data architect of Graphwise/Ontotext
- Original location: https://github.com/VladimirAlexiev/RailDataForum2025-SPARQL
- Mirrored to: https://gitlab.com/era-europa-eu/public/interoperable-data-programme/era-ontology/rail-data-forum-2025/sparql-for-beginners-masterclass
- Last updated: 2025-07-08
  - 18-Jun-2025: Wrote blog post [Improving ERA SHACL](improving-era-shacl.md) in this same repository.
  - 3-Jul-2025: Added section  [Visual Graph](#visual-graph)

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [RailDataForum2025 SPARQL Tutorial](#raildataforum2025-sparql-tutorial)
    - [Intro](#intro)
        - [Citation](#citation)
        - [SPARQL eLearning](#sparql-elearning)
        - [ERA KG Intro](#era-kg-intro)
        - [ERAbot](#erabot)
        - [SPARQL Intro](#sparql-intro)
    - [Some SPARQL Queries](#some-sparql-queries)
        - [Longest Tunnel](#longest-tunnel)
        - [Tunnels in Romania](#tunnels-in-romania)
        - [Names of Romania](#names-of-romania)
        - [Collect Names](#collect-names)
        - [Ordered and Distinct Names](#ordered-and-distinct-names)
        - [Query Explanation by GPT](#query-explanation-by-gpt)
    - [Schema Exploration](#schema-exploration)
        - [Class Count](#class-count)
        - [ERA Class Count](#era-class-count)
        - [Property Count](#property-count)
        - [ERA Property Count](#era-property-count)
        - [Deprecated Terms](#deprecated-terms)
        - [SKOS Vocabularies](#skos-vocabularies)
    - [Complex Queries](#complex-queries)
        - [Canonical URIs](#canonical-uris)
        - [Data Duplication](#data-duplication)
        - [Counting Tunnels](#counting-tunnels)
    - [Visual Graph](#visual-graph)
    - [Data Stories](#data-stories)

<!-- markdown-toc end -->

## Intro

[RailDataForum2025](https://www.era.europa.eu/agenda/agenda-rail-data-forum-2025) offers 3 "SPARQL for Beginners" sessions:
- June 11 9:30-10:30 and June 12 13:30-15:15 (a longer session of 1:45h).
- I assume some basic knowledge of the RDF graph model and the Turtle format (serialization),
  since SPARQL patterns are based on the same syntax.
- You can follow this by a "Practical Data Consumption" session that will be led by Ghislain Atemezing of ERA.

We will explain the basics of SPARQL, 
show some queries (competency questions), 
an `ERAbot` chatbot that can generate SPARQL,
and some graph visualizations.

### Citation
If you use this material, please cite it as:
```
@Misc{Alexiev-RailDataForum-SPARQL-2025,
  author       = {Vladimir Alexiev},
  title        = {RailDataForum2025: SPARQL Masterclass},
  howpublished = {Tutorial at RailDataForum2025},
  month        = jun,
  year         = 2025,
  url          = {https://github.com/VladimirAlexiev/RailDataForum2025-SPARQL},
  keywords     = {rail, ERA, ERA Vocabulary, ERA KG, SPARQL, tutorial, RailDataForum, Talk to Your Graph, TTYG, chatbot, ERAbot},
  abstract     = {A SPARQL tutorial presented at the RailDataForum2025 masterclass. We present sample queries, schema exploration queries, and link to the ERA Vocabulary "data stories" (competency questions). We also introduce the ERAbot, setup quickly by using GraphDB's "Talk to Your Graph" functionality. Contents: Intro: SPARQL eLearning, ERA KG Intro, ERAbot, SPARQL Intro; Some SPARQL Queries: Longest Tunnel, Tunnels in Romania, Names of Romania, Collect Names, Ordered and Distinct Names, Query Explanation by GPT; Schema Exploration: Class Count, ERA Class Count, Property Count, ERA Property Count, Deprecated Terms, SKOS Vocabularies Complex Queries: Canonical URIs, Data Duplication, Counting Tunnels; Data Stories},
}
```

If you like the "Improving ERA SHACL" post, cite it as:
```
@Misc{Alexiev-ERA-SHACL-2025,
  author       = {Vladimir Alexiev},
  title        = {Improving ERA SHACL},
  howpublished = {Blog post},
  month        = jun,
  year         = 2025,
  url          = {https://github.com/VladimirAlexiev/RailDataForum2025-SPARQL/blob/main/improving-era-shacl.md},
  keywords     = {rail, ERA, ERA Vocabulary, ERA KG, SHACL, RDF shapes, data validation, SPARQLConstraint, SPARQLTarget},
  abstract     = {The ERA KG is the result of a large-scale collaboration between Europe's rail entities and is the foundation of EU rail data interoperability. It is complex and relatively big, and SHACL data validation is used to improve data quality. In this blog we describe Graphwise experience with complex SHACL shapes, show how ERA SHACL shapes can be simplified by a third, how their performance can be optimized (instead of million times, run SPARQL once), and discuss incremental validation scenarios that can further improve performance. In particular, section Constraints for SKOS Properties shows how to replace 73 to 103 enum constraints that run SPARQL over 8M times (per enum prop instance); with 1 constraint that runs SPARQL once (then a few cheap SPARQLs per violation)},
}
```

### SPARQL eLearning

The [Ontotext Academy](https://academy.ontotext.com/welcome.html) has a "GraphDB Knowledge Graph Engineer" learning path.
Note: GraphDB is replacing Virtuoso as the semantic database of the ERA KG, 
but we use only standard SPARQL 1.1, so your skills will be transferable.
The academy is free, you just need to register with your email.

That learning path has 3 courses:
- Course 1: Introduction to Semantic Technologies
- Course 2: Semantic models with GraphDB
- Course 3: SPARQL

Normally the 3 courses need to be taken in sequence, and we highly recommend it if you have no experience with semantic technologies
or want to get a certificate.

If you want to take only the SPARQL course, please register and then contact [bob.ducharme@graphwise.ai](mailto:bob.ducharme@graphwise.ai).
Bob is the author of the famous [Learning SPARQL](https://www.learningsparql.com/) book that you can buy on [Amazon](https://www.amazon.com/Learning-SPARQL-Querying-Updating-1-1/dp/1449371434/) in paper or Kindle format.

### ERA KG Intro

- Virtuoso
  - [SPARQL editor](https://data-interop.era.europa.eu/endpoint): uses YasGUI, has syntax highlighting and autocompleting
  - [SPARQL endpoint](https://data-interop.era.europa.eu/api/sparql): API for posting queries
- GraphDB
  - http://rail.sandbox.ontotext.com/sparql : SPARQL editor. User `rdf2025`, password `Gr@phwise2025`
  - This is slightly older than the Virtuoso endpoint as it uses the ERA KG mentioned next; 
    but it has a chatbot.
- [ERA KG on Zenodo](https://zenodo.org/records/14605744) (Jan 6, 2025)
- [RINF data stories](https://data-interop.era.europa.eu/data-stories) (competency questions, see [source](https://github.com/Interoperable-data/ERA_vocabulary/tree/main/queries)): 
  you can select and execute them directly in Virtuoso, or copy from source and execute in GraphDB
- [Current ontology 3.0.1](https://data-interop.era.europa.eu/era-vocabulary/v3-20240618/) of 2024-06-18 (see [source](https://github.com/Interoperable-data/ERA_vocabulary)):
  - [Ontology diagram](https://data-interop.era.europa.eu/era-vocabulary/v3-20240618/#desc), reproduced below
  - [SKOS thesauri](https://data-interop.era.europa.eu/era-vocabulary/v3-20240618/skos/index.html) (enumeration values)
  - Note that ERA KG also uses SKOS taxonomies from the Publications Office,
    in particular http://publications.europa.eu/resource/authority/country
- [Future ontology 3.1.1](https://data-interop.era.europa.eu/era-vocabulary/) of 2025-05-12 (see [source](https://gitlab.com/era-europa-eu/public/interoperable-data-programme/era-ontology/era-ontology))
  - (Infrastructure Managers have a year to start submitting RINF data in accordance with this new version)

Ontology diagram of version 3.0.1:

![](https://data-interop.era.europa.eu/era-vocabulary/v3-20240618/resources/images/ERA-classes-overview.png)

We'll work with these main objects:
- `OperationalPoint`: any location for train service operations, or any location at boundaries between Member States or infrastructure managers. Includes station, stop, switch point, etc
- `SectionOfLine`: part of line between adjacent operational points; may consist of several tracks; cannot diverge
- `Tunnel`: a railway tunnel

You can learn more by reading the ontology documentation, but mind you it's pretty big!

### ERAbot

http://rail.sandbox.ontotext.com/ttyg is a GraphDB "Talk to Your Graph" chatbot.
User `rdf2025`, password `Gr@phwise2025`
- Select `ERAbot` and ask it some questions about the ontology, taxonomies, or instance data.
- Then use `Explain Response` to see how the bot came up with an answer, and what query it used.
- You can also click the two buttons on top right of the query to load it in the SPARQL editor, or copy it

![](ERAbot-longest-tunnel-Europe.png)

Rather than writing SPARQL, use the bot to do it for you :-)
And I'll try to teach you to read, spot errors in, and debug SPARQL.


### SPARQL Intro

SPARQL is a set of specifications (see [overview](https://www.w3.org/TR/sparql11-overview/)) that includes:
- Query language
- Update language
- Query protocol, including parameterization
- Result formats:
  - Tabular: SPARQL Results XML, SPARQL Results JSON, CSV, TSV
  - RDF: RDF/XML, Turtle, JSON-LD, NTriples, etc
- Federated Querying, for making queries across repositories
- Service Description, for discovering the features of a particular repository
- SPARQL HTTP Graph Store Protocol, for manipulating RDF graphs with simple GET/PUT/POST operations

The current version of SPARQL is 1.1.
(If you are curious about ideas and possible future enhancements to SPARQL, 
see https://github.com/w3c/sparql-dev ).

Once you learn SPARQL a bit, use [SPARQL 1.1 Syntax Diagrams](https://vladimiralexiev.github.io/grammar-diagrams/sparql11-grammar.xhtml) for reference.
It presents cross-linked SPARQL 1.1 syntax (railroad) diagrams, one per production
EBNF syntax rules were extracted from the SPARQL 1.1 specification (Query and Update).

SPARQL Query has 4 query forms:

![](SPARQL-diagram-top.png)

- SELECT for returning a table of results (in one of tabular formats).
- CONSTRUCT for returning a constructed subgraph (in one of the RDF formats).
  It uses the same evaluation algebra as SELECT, 
  and then substitutes results in a CONSTRUCT template.
- DESCRIBE for returning information about one or several resources (nodes, URIs) (in one of the RDF formats).
  It returns the Symmetric Concise Bounded Description, i.e. incoming and outgoing triples,
  and navigates over blank nodes (on the assumption that they are "owned" by the resource).
- ASK for checking the existence of some graph pattern, or liveness of a SPARQL endpoint (YES/NO result).

In this tutorial we'll work only with SELECT.
It's a complex syntax, and here are the top-level diagrams:

![](SPARQL-diagram-select.png)

## Some SPARQL Queries

Ok, let's make some queries!
Along the way I'll share advice on how to avoid common pitfalls.

### Longest Tunnel

Rather than a toy query, let's start with the query shown above:
```sparql
PREFIX : <http://data.europa.eu/949/>
PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
SELECT ?tunnel ?label ?length ?countryName WHERE {
  ?tunnel a :Tunnel ;
          rdfs:label ?label ;
          :lengthOfTunnel ?length ;
          :inCountry ?country .
  ?country skos:prefLabel ?countryName .
  FILTER(LANG(?countryName) = "en")
} ORDER BY DESC(?length) LIMIT 1
```

Let's explain it line by line:
- The query starts with some prefixes, which allow you to use short URLs in the query
  - `:` is an empty prefix for the main ontology (ERA). You can also see it written as `era:`
  - `skos:` is the ontology used to represent thesauri
  - You may notice that the `rdfs:` prefix is missing
    - GraphDB workbench has a prefix auto-insertion feature:
      the prefix is added when you paste a query, or type the colon at the end of the prefix
    - For that reason, I won't bother to give prefixes in the queries below
    - The chatbot has a setting to add missing prefixes
    - But before running the query all prefixes must be defined.
- The SELECT clause specifies which variables to return and in what order
  - Variables are denoted as `?var` or `$var`
  - I often like ot use `select *` to return all vars, in order to not mask over-selection problems
    (will explain later).
    I also care to order the variables in the way I want them (from the most important to less important)
- The body of the query is a graph pattern, formatted as in Turtle.
  We can use fullstop to separate subjects, 
  semicolon to separate props of the same subject, 
  or comma to separate values of the same subject-property pair.
  Patterns can also use variables
- The patterns are followed by a `filter`.
  `?country` comes from a Publication Office thesaurus of countries, 
  and it has `prefLabel` in a variety of languages.
  We want each country to be considered only once, so we require the `lang()` of the label to be English
- At the end is a Solution Modifier that orders all results by descending length (`DESC()`) 
  and takes only the top one.

### Tunnels in Romania

Given the previous query, it's easy to find the top 3 tunnels in Romania:
```sparql
SELECT * {
  ?tunnel a :Tunnel ;
          rdfs:label ?label ;
          :lengthOfTunnel ?length ;
          :inCountry ?country .
  ?country skos:prefLabel "Romania"@en
} ORDER BY DESC(?length) LIMIT 3
```

Here we use the exact label of the country `"Romania"@en` (that is an `rdf:langString`).
We could have used eg the Portuguese name `"Roménia"@pt` with the same effect.

### Names of Romania

The Countries thesaurus knows the names of Romania in 43 languages
(there are names in Chinese, Japanese and Korean!)

```sparql
select * {
  ?country skos:prefLabel "Romania"@en, ?label
}
```
- The first pattern `?country skos:prefLabel "Romania"@en` finds the country we want
- The second pattern (can be expanded to) `?country skos:prefLabel ?label` returns all its labels

BTW the full URL is `http://publications.europa.eu/resource/authority/country/ROU`
and uses the ISO 3-letter code of the country.

This is displayed shortened as `atold:country/ROU` 
where `atold` means (I guess) "autorities as linke data".
However, becuase of the `/` this is not a proper prefixed name, it's a Compact URI (CURIE).
You cannot use it in a query, but it's used to present results in a shorter form.

### Collect Names

I'm too lazy to show the result table of 43 names, so let's collect them as one string:
```sparql
select ?country (group_concat(?label) as ?labels) {
    ?country skos:prefLabel "Romania"@en, ?label
} group by ?country
```

> Romania Romania Romania Romania Romania Rúmenía Ir-Rumanija Romanía Rumānija Roménia Rumänien Rumänien Rumänien Румунія Rumenia Rumenia An Rómáin Románia Rumænien Roumanie Rumunia Rumunsko Rumunsko Rumeenia Rumanía Errumania Rumunija Rumunija Румынія România ルーマニア Rumania Ρουμανία Roemenië Rumunjska 罗马尼亚 Румыния رومانيا Romunija Pyмъния Румунија Romanya Романија

We introduce several new things here:
- The `GROUP BY` solution modifier lists a few variables, and applies **aggregation** to the other variables
- `group_concat()` is one of the aggregations. 
  It concatenates the values of `?label` grouped per `?country` (in this case there's only 1)
- `(<expression> as ?var)` calculates an expression and binds it to `?var` right in the SELECT.
  You can do the same in the body by using `bind()`

BTW you probably noticed that SPARQL keywords are case-insensitive.
Some people like to write them in uppercase, 
but I prefer lowercase so as not to draw attention away from the complex parts of the query.

### Ordered and Distinct Names

The string "Romania" appears several times above. 
The reason is that the same string is used as name in several languages (e.g. all of the Latin languages).

Suppose we want to find only the distinct string values.
It'd also be nice to order them, in order to more easily check that there are no duplicates:

```sparql
select ?country (group_concat(distinct ?l; separator="; ") as ?labels) {
  {select ?country ?l { # could use DISTINCT here
    ?country skos:prefLabel "Romania"@en, ?label
    bind(str(?label) as ?l)
  } order by ?l}
} group by ?country 
```

We introduce several new mechanisms here:
- `DISTINCT` eliminates duplicates from a result. You can use it in SELECT, or inside `group_concat()`.
- We use `separator=";"` to more clearly see the different names, because one them includes a space
- We use `str(?label)` to convert the `langString` to a simple `string` 
  because otherwise the ordering will be first by lang tag, and then by value
- We use a subquery to enforce the ordering: simply using `order by` on the outer query doesn't work.
  - You may wonder how did I know that: I didn't! 
    I developed the query incrementally, which is a very common approach
  - Subqueries are wrapped in `{...}` and must select the variables to be exported outside of the subquery.
  - They are also used to enforce an order of execution, especially in federated queries

> An Rómáin; Errumania; Ir-Rumanija; Pyмъния; Roemenië; Romania; Romanya; Romanía; Romunija; Románia; România; Roménia; Roumanie; Rumania; Rumanía; Rumeenia; Rumenia; Rumunia; Rumunija; Rumunjska; Rumunsko; Rumänien; Rumænien; Rumānija; Rúmenía; Ρουμανία; Романија; Румунија; Румунія; Румыния; Румынія; رومانيا; ルーマニア; 罗马尼亚

### Query Explanation by GPT

You can use `Alt-Ctrl-Enter` (or `Alt-Cmd-Enter` on Mac) in GraphDB workbench to run the query, 
then explain the query and result using ChatGPT.
In case you want to access this programmatically,
the explanation is returned in an extra row of the resultset in a variable `_gpt`.

Below is the explanation from GPT. 
It is pretty detailed and informative but doesn't explain some finer points,
e.g. **why** an inner query is needed.

----

The provided SPARQL query retrieves information about the country "Romania" from a dataset that uses the SKOS (Simple Knowledge Organization System) vocabulary. The query performs the following:

1. **Namespace Declaration:** The `PREFIX` line specifies that `skos:` refers to the SKOS namespace, which is used for accessing SKOS properties like `prefLabel`.

2. **Data Selection:**
   - The query looks for a resource (`?country`) that has a preferred label (`skos:prefLabel`) of "Romania" in English (`"Romania"@en`). It implies that ?label can be any other label in different languages or variations that describe Romania.
   - The `BIND` function is used to convert these labels to a string and store them in variable `?l`.

3. **Inner Query:** 
   - An inner query retrieves all possible labels for the country identified as "Romania", ordering them by their string values.

4. **Grouping and Concatenation:**
   - The outer query groups the results by the `?country` variable and uses `group_concat` to combine all distinct labels into a single string separated by "; ".

5. **Result Explanation:**
   - The result shows a single row with the `?country` being `<http://publications.europa.eu/resource/authority/country/ROU>`, which is a URI that identifies Romania according to an authority list from the EU.
   - `?labels` lists all the distinct labels concatenated for Romania, presented in various languages or transliterations. These include, among others, "Romania" in English, "România" in Romanian, and translations/transliterations in languages like German, Spanish, French, Greek, Russian, Japanese, Chinese, etc.

Essentially, the query and result demonstrate how Romania is represented and labeled in different languages and schemas across a multilingual dataset using SKOS concepts."

## Schema Exploration

Given that RDF schemas (ontologies and SHACL shapes) are also represented as RDF,
it should be no surprise that we can introspect and explore them
(in this tutorial we work only with ontologies).

### Class Count

This is a "standard" query to count classes by occurrence.
It is often used as a first step in exploring an unknown repository.

```sparql
select ?x (count(*) as ?c) {
    [] a ?x
} group by ?x order by desc(?c)
```
Explanation
- I often use `?x` as the main thing I'm querying for, and `?c` for a count
- `[]` is a blank node, which means here "any subject, but I don't care about it"
- `a` is an abbreviation for `rdf:type`

You may want to write this as a more telling pattern `?resource rdf:type ?class`,
but for me brevity is a virtue.

We order the results by descending count of instances.
- The most popular classes are `NetElement, NetRelation, Track` (364k), `TrainDetectionSystem` (184k).
  - Curiously, the 2 most popular classes are not defined, I have filed a bug.
- There are 64k `wgs:Point` (simple coordinate pairs), also represented as `geo:Geometry` 
  (GeoSPARQL WKT representation)
- There are 34 classes, of which about 24 are domain-oriented (ERA-specific): see next
- There are 2170 `Concepts` (thesaurus values) in 77 `ConceptSchemes` (thesauri): see further below

### ERA Class Count

Let's count only ERA classes by number of instances.
There are several ways to do it:
- by the link `rdfs:isDefinedBy` or by prefix `era:`
- Through instances or only from the definition of terms as `a owl:Class`

Let's try:
- By instances and link: 18 classes
```sparql
select ?x (count(*) as ?c) {
    [] a ?x.
    ?x rdfs:isDefinedBy era:
} group by ?x order by desc(?c)
```
- By instances and namespace: 20 classes (I already mentioned that `NetElement, NetRelation` are not defined).
  We use the `strstarts()` function to check that the `era:` URI (namespace) is a prefix of the class URI
```sparql
PREFIX era: <http://data.europa.eu/949/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
select ?x (count(*) as ?c) {
    [] a ?x.
    filter(strstarts(str(?x),str(era:)))
} group by ?x order by desc(?c)
```
- by definition and link: 35 classes.
```sparql
select ?x {
    ?x a owl:Class.
    ?x rdfs:isDefinedBy era:
} order by ?x
```
- by definition and namespace: 64 classes. This means that:
  - 29 classes lack the `isDefinedBy` link (reported as [era-ontology issue 113](https://gitlab.com/era-europa-eu/public/interoperable-data-programme/era-ontology/era-ontology/-/issues/113))
  - Only 1/3 of all defined classes are used in the data (this is ok, the KG has space to grow after the ontology)
```sparql
select ?x {
    ?x a owl:Class
    filter(strstarts(str(?x),str(era:)))
} order by ?x
```

### Property Count
This query counts properties by instance:
```sparql
select ?x (count(*) as ?c) {
    [] ?x []
} group by ?x order by desc(?c)
```
- `[] ?x []` means "Find `?x` in property position, but I don't care about the subject or object"

The most populated props are:
- `era:notApplicable` (6.27M) and `era:notYetAvailable` (5.3M): 
  this is used to describe for a rail resource, which expected properties are not there, and why
- `rdf:type` (6.23M): this is approximately the number of instances 
  (but can exceed it, since a resource may have multiple types)

A total of 320 props are instantiated (populated).

### ERA Property Count

Now let's check how many ERA-specific props are defined and instantiated (populated)

- Defined by link: 469. 
  - Each prop is defined as datatype or object property, but not both.
    We use a `values` list to specify these two "kinds":
```sparql
PREFIX owl: <http://www.w3.org/2002/07/owl#>
PREFIX era: <http://data.europa.eu/949/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
select ?x {
    values ?kind {owl:DatatypeProperty owl:ObjectProperty}
    ?x a ?kind.
    ?x rdfs:isDefinedBy era:
} order by ?x
```
- Defined by prefix: 501. This means that 32 props lack the `isDefinedBy` link 
  (reported as [era-ontology issue 113](https://gitlab.com/era-europa-eu/public/interoperable-data-programme/era-ontology/era-ontology/-/issues/113))
```sparql
select ?x {
    values ?kind {owl:DatatypeProperty owl:ObjectProperty}
    ?x a ?kind.
    filter(strstarts(str(?x),str(era:)))
} order by ?x
```
- Instantiated by link: 228 props
  - Here we use a new construct `filter exists`.
    We first find candidates by `rdfs:isDefinedBy era:` (these are classes and props),
    then filter to only those that are actually used as props.
  - It makes the query fast because for each candidate, only one matching `[] ?x []` needs to be examined
```sparql
select ?x {
    ?x rdfs:isDefinedBy era:
    filter exists {[] ?x []}
} order by ?x
```
- Instantiated by prefix: 235 props
  - This query takes the longest (38sec) because DISTINCT is an expensive operation:
    It has to find all results `?x`, put them in memory, sort and compare them to eliminate duplicates
```sparql
select distinct ?x {
    [] ?x []
    filter(strstarts(str(?x),str(era:)))
} order by ?x
```

### Deprecated Terms

Like any large ontology that has evolved for a while, the ERA Vocabulary includes some deprecated terms.
These are not deleted for existing data, but should not be used for future data, 
and should be migrated gradually.
The standard Boolean property `owl:deprecated` is used as such a flag:

```sparql
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX owl: <http://www.w3.org/2002/07/owl#>
select * where {
   ?x owl:deprecated true.
   optional {?x dct:isReplacedBy ?y}
} order by ?x
```

ERA has 91 deprecated terms. 
About 20 of them also point to a new (replacement) term using `dct:isReplacedBy`.
Note how we use `optional {...}` to also return results that lack that information.

E.g. `era:frenchTrainDetectionSystemLimitation` is replaced by 3 potential terms:
- `era:frenchTrainDetectionSystemLimitationApplicable`
- `era:frenchTrainDetectionSystemLimitationNumber`
- `era:tdsFrenchTrainDetectionSystemLimitation`

This query also returns about 40 obsolete Publication Office country records:
e.g. `AFI, ANT, ATN, BUR` etc are obsolete.

### SKOS Vocabularies

The ERA Vocabulary uses 67 thesauri represented as `skos:ConceptScheme`
(this query also returns about 10 schemes from the Publications Office countries authority list):
```sparql
select * {
    ?x a skos:ConceptScheme
} order by ?x
```
Thesaurus entries (enumeration values) are represented as `skos:Concept`.

According to SKOS modeling principles, each concept should be in a scheme, and that is indeed satisfied:
this returns nothing:
```sparql
select * {
    ?x a skos:Concept.
    filter not exists {?x skos:inScheme ?y}
}
```

Some taxonomies have a hierarchical structure represented with `skos:broader`.
The roots of each hierarchy have a link `skos:topConceptOf` to the scheme. Ther are 2883 such roots:
```sparql
select * {
    ?x skos:topConceptOf ?y
}
```

And all of them have the mandatory link `inScheme` (this query returns nothing):
```sparql
select * {
    ?x skos:topConceptOf ?y
    filter not exists {?x skos:inScheme ?y}
}
```

With these preliminary checks, let's count the number of concepts per scheme:
```sparql
select ?x (count(*) as ?c) {
    [] skos:inScheme ?x
} group by ?x order by ?x
```
| x                                                                            |   c |
|------------------------------------------------------------------------------|-----|
| concepts/axle-monitoring/AxleBearingMonitoring                               |  25 |
| concepts/brake-parking-type/BrakeParkingType                                 |   4 |
| concepts/company-code-categories/CompanyCodeCategories                       |   3 |
| concepts/compliant-pantograph-heads/CompliantPantographHeads                 |   5 |
| concepts/contact-line-systems/ContactLineSystems                             |   4 |
| concepts/contact-strip-materials/ContactStripMaterials                       |  54 |
| concepts/eddy-current-braking/EddyCurrentBraking                             |   5 |
| concepts/end-coupling-type/EndCouplingType                                   |  77 |
| concepts/energy-supply-systems/EnergySupplySystems                           |  36 |
| concepts/etcs-baselines/ETCSBaselines                                        |   8 |
| concepts/etcs-equipment-on-board-level/ETCSEquipmentLevels                   |  20 |
| concepts/etcs-infills/ETCSInfills                                            |   7 |
| concepts/etcs-levels/ETCSLevels                                              |   6 |
| concepts/etcs-m-versions/ETCSMVersions                                       |   4 |
| concepts/etcs-situation/ETCSSituations                                       |   6 |
| concepts/etcs-system-compatibilities/ETCSSystemCompatibilities               | 149 |
| concepts/freight-corridor/FreightCorridors                                   |  11 |
| concepts/gauge-changeover-facilities/GaugeChangeoverFacilities               |  10 |
| concepts/gaugings/GaugingProfiles                                            | 136 |
| concepts/gsm-r-equipment-version/GSMREquipmentVersion                        |  41 |
| concepts/gsm-r-radio-data-communication/GSMRRadioDataCommunications          |  14 |
| concepts/gsmr-networks/GSMRNetworks                                          |  56 |
| concepts/gsmr-number-active-mobiles/NumberActiveMobiles                      |   3 |
| concepts/gsmr-optional-functions/OptionalFunctions                           |  18 |
| concepts/gsmr-versions/GSMRVersions                                          |  24 |
| concepts/hot-axle-box-detector-direction/HotAxleBoxDetectorDirections        |   3 |
| concepts/ice-conditions/IceConditions                                        |  43 |
| concepts/legacy-radio-systems/LegacyRadioSystems                             | 151 |
| concepts/line-category/LineCategories                                        |  14 |
| concepts/load-capabilities/LoadCapabilities                                  | 212 |
| concepts/load-capability-line-categories/LoadCapabilityLineCategories        |  22 |
| concepts/magnetic-braking/MagneticBraking                                    |   5 |
| concepts/max-amount-sandings/MaxAmountSandings                               |   4 |
| concepts/min-axle-load-vehicle-categories/MinAxleLoadVehicleCategories       |   3 |
| concepts/min-axle-loads-per-vehicle-category/MinAxleLoadsPerVehicleCategory  |  10 |
| concepts/navigabilities/Navigabilities                                       |   4 |
| concepts/nominal-track-gauges/NominalTrackGauges                             |  14 |
| concepts/op-types/OperationalPointTypes                                      |  15 |
| concepts/osm-classes/OSMClasses                                              |   8 |
| concepts/other-pantograph-heads/OtherPantographHeads                         |   9 |
| concepts/other-protection-control-warning/OtherProtectionControlWarnings     |  48 |
| concepts/platform-heights/PlatformHeights                                    | 108 |
| concepts/profile-num-semi-trailers/ProfileNumbersSemiTrailers                |  36 |
| concepts/profile-num-swap-bodies/ProfileNumbersSwapBodies                    |  36 |
| concepts/radio-system-compatibilities-data/RadioSystemCompatibilitiesData    |  42 |
| concepts/radio-system-compatibilities-voice/RadioSystemCompatibilitiesVoice  |  22 |
| concepts/rail-inclinations/RailInclinations                                  |  24 |
| concepts/relation-positions/RelationPositions                                |   2 |
| concepts/restrictions/Restrictions                                           |   7 |
| concepts/rolling-stock-fire/Categories                                       |   9 |
| concepts/sol-natures/SoLNatures                                              |   2 |
| concepts/states/States                                                       |   4 |
| concepts/temperature-ranges/TemperatureRanges                                |   4 |
| concepts/ten-classifications/TENClassifications                              |   6 |
| concepts/thermal-capacity/ThermalCapacityTSIReferences                       |   4 |
| concepts/track-running-directions/TrackRunningDirections                     |   3 |
| concepts/train-detection-numbers/FrenchTrainDetectionSystemLimitationNumbers |   8 |
| concepts/train-detection-specific-checks/TrainDetectionSystemsSpecificChecks |  14 |
| concepts/train-detection/FrenchTrainDetectionSystemLimitations               |  17 |
| concepts/train-detection/TrainDetectionSystems                               |   7 |
| concepts/train-protection-legacy-systems/TrainProtectionLegacySystems        | 160 |
| concepts/tsi-compliances/TSICompliances                                      |   2 |
| concepts/tsi-existence-and-compliances/TSIExistenceAndCompliances            |   3 |
| concepts/type-version-ids/TypeVersionIds                                     |   5 |
| concepts/vehicle-types/Categories                                            |   4 |
| concepts/vehicle-types/SubCategories                                         |  15 |
| http://publications.europa.eu/resource/authority/country                     | 341 |

Note: I've omitted all but one Country-related taxonomy.

Each enumeration property is declared `owl:ObjectProperty` with `rdfs:range skos:Concetp`,
and the ontology has an extra prop that points to the concept scheme: 
```sparql
select * {
   ?p :inSkosConceptScheme ?s
} order by ?p
```

There are 103 such props, which means in a few cases several props share the same concept scheme.
Let's find them out:
```sparql
PREFIX : <http://data.europa.eu/949/>
select * {
   ?p :inSkosConceptScheme ?s
   filter exists {
     ?p1 :inSkosConceptScheme ?s
     filter (?p != ?p1)
   }
} order by ?s
```

- All these 4 props use the Countries taxonomy: 
  `era:authorizedCountry, era:inCountry, era:manufacturingCountry, era:quieterRoutesExemptedCountry`
- Another example are these 2 props, which both use `concepts/track-running-directions/TrackRunningDirections`:
  `era:fireSafetyCategory, era:rollingStockFireCategory`

## Complex Queries

In this section we investigate some more complex topics

### Canonical URIs

This concept is a bit hard to undertstand, but fundamental for understanding ERA KG data organization:

> The canonical URI is defined for each instance of an Infrastructure element, e.g. section of line, operational point, track, tunnel, siding.
> Objects of the infrastructure generated through RML mappings include (when provided) their validity start and end dates. With its identifier, plus all identifiers of its "parent" elements, and its validity dates, a hash URI is generated.
> The canonical URI is the element's URI with its identifiers and without the validity dates. All of the hash URIs of an element point to its canonical URI.

This means that
- A rail element is represented by a different resource whenever some of its attributes change,
and distinct `era:validityStartDate, era:validityEndDate` are recorded.
- A hash is computed from all variable elements, and they are recorded in field `era:hashSource`
- All these "sibling" resources share the same `era:canonicalURI`, 
  which in a sense is the "true identifier" of the resource

Let's find the canonical URI with most representatives, i.e. the element with most variant resources:
```sparql
select ?uri (count(*) as ?c) {
    ?x :canonicalURI ?uri
} group by ?uri order by desc(?c) limit 3
```

http://data.europa.eu/949/functionalInfrastructure/tunnels/S-Bahn-Tunnel%20Frankfurt%20City_%2B8.66282650.107369_%2B8.68875950.100673
has 50 variants.

Now let's examine all properties of these resources, and count the unique values:
```sparql
select ?p (count(distinct ?o) as ?c) {
  ?x :canonicalURI <http://data.europa.eu/949/functionalInfrastructure/tunnels/S-Bahn-Tunnel%20Frankfurt%20City_%2B8.66282650.107369_%2B8.68875950.100673>.
  ?x ?p ?o
} group by ?p order by ?p
```
| p                         |  c |
|---------------------------|----|
| :canonicalURI             |  1 |
| :endLocation              |  1 |
| :hashSource               | 50 |
| :imCode                   |  1 |
| :inCountry                |  1 |
| :length                   |  1 |
| :lengthOfTunnel           |  1 |
| :lineReferenceTunnelEnd   |  8 |
| :lineReferenceTunnelStart |  8 |
| :netElement               | 25 |
| :notYetAvailable          |  4 |
| :rollingStockFireCategory |  2 |
| :startLocation            |  1 |
| :tunnelIdentification     |  1 |
| :validityEndDate          |  2 |
| :validityStartDate        |  2 |
| rdf:type                  |  1 |
| rdfs:label                |  1 |

The most important props (`type, label, start/endLocation, tunnelIdentification, inCountry`) are constant.
What changes are the finer details. In particular:
- `:lineReferenceTunnelStart, :lineReferenceTunnelEnd` 
  vary between 8 values incorporating "single-track, directional track, opposite track" and more
- `:validityStartDate, :validityEndDate` vary between 2024, 2025, 2026 (whole years)
- The hash always varies. One hash of that tunnel is below.
  As you see, it includes dates, line references, and geographic coordinates

`3610_DE00FFT_DE000FF/2024-01-01_2024-12-31/directional track/S-Bahn-Tunnel Frankfurt City_+8.66282650.107369_+8.68875950.100673/2024-01-01_2024-12-31`

### Data Duplication

What is `era:Tunnel` (or `:Tunnel`)? 
Despite the definition, it's not a real railway element, but a version thereof.
(Read the previous section to see why).

You should take this into account when doing aggregations.
E.g. let's try to find the tunnels of Switzerland, ordered by length:
```sparql
select * {
  [] a :Tunnel;
      :inCountry/skos:prefLabel "Switzerland"@en;
      rdfs:label ?name;
      :lengthOfTunnel ?len
} order by desc(?len)
```
Here we use a couple more tricks:
- `[]` is the `:Tunnel` URI but since it's uninteresting, I write it as a blank node `[]`.
  This lets me write `select *` and not worry I've missed a variable I add to the query body
- `:inCountry/skos:prefLabel` is a sequence property path. 
  It goes across the country node, and examines its label.

> ASIDE: you should use the SPARQL editor autocompletion to your advantage.
> Eg I typed `:tunnelL` then pressed control-space since I wasn't sure of the spelling.
> The autocomplete breaks this camelCase word into multiple words and searches for them separately.
> So it was able to find the correct prop: `:lengthOfTunnel`.

This returns 251 tunnels. Sure, Switzerland is mountainous, but does it have so many tunnels?
If you examine the result, you find lots of data duplication:
- Gotthard-Basistunnel (57363m) listed 4 times
- Simplontunnel (19820) listed 6 times
- etc

Adding `select distinct *` fixes the problem for this query, though I have an uneasy feeling about it:
it relies on all selected details of the variant resources to be spelled exactly consistently,
which is a strong assumption.

### Counting Tunnels

Let's count the number of tunnels per country, their total and average length
```sparql
select ?country
       (count (*) as ?c) 
       (xsd:integer(sum(?len)) as ?length) 
       (xsd:integer(sum(?len)/?c) as ?average) {
    {select distinct * {
        [] a :Tunnel;
          :inCountry/skos:prefLabel ?country;
          rdfs:label ?name;
          :lengthOfTunnel ?len
        filter(lang(?country)="en")}}
} group by ?country order by ?country
```
- The inner query does `distinct` over the variables `?country ?name ?length` 
  to eliminate the duplicates described in the previous section
- We convert the total and average length to integers, since we don't need multi-digit precision

The result:

| country     |    c |  length | average |
|-------------|------|---------|---------|
| Austria     |  191 |  234937 |    1230 |
| Belgium     |  161 |  103826 |     644 |
| Bulgaria    |  145 |   36078 |     248 |
| Croatia     |   52 |   21075 |     405 |
| Czechia     |  159 |   54855 |     345 |
| Denmark     |   42 |   36004 |     857 |
| Finland     |    3 |    8871 |    2957 |
| France      |  916 |  531719 |     580 |
| Germany     |  627 |  571451 |     911 |
| Greece      |  118 |   74599 |     632 |
| Hungary     |   19 |   11631 |     612 |
| Italy       | 1609 | 1505113 |     935 |
| Lithuania   |    1 |    1285 |    1285 |
| Luxembourg  |   26 |    6221 |     239 |
| Netherlands |   45 |   74248 |    1649 |
| Norway      |  466 |  301616 |     647 |
| Poland      |   37 |   22081 |     596 |
| Portugal    |   50 |   24020 |     480 |
| Romania     |   46 |   14357 |     312 |
| Slovakia    |   47 |   30877 |     656 |
| Slovenia    |   85 |   37373 |     439 |
| Spain       | 1930 | 1366144 |     707 |
| Sweden      |  170 |  146620 |     862 |
| Switzerland |   42 |  151170 |    3599 |

If you sort by the different columns, you find some interesting facts:
- Bulgaria has 3.5x more tunnels than Switzerland! That wass hard for me to believe, so I checked.
- Lithuania has a single tunnel, and Finland has 3. But because they are pretty long,
  these are amongst the 4 top countries by average tunnel length
- Spain has 1930 tunnels, totaling 1366 km !
- Italy has a bit fewer (1606) but is the leader of total tunnel length: 1505 km !

## Visual Graph

GraphDB can make charts from the result of SELECT queries, and visual graphs from the result of CONSTRUCT queries.
Let's try the latter to make a graph of Operational Points in Bulgaria.
I saved it as an "Advanced Graph Configuration" that you can [edit here](https://rail.sandbox.ontotext.com/graphs-visualizations/config/save/8839d5b2864e47a4ad26d74f1f0423a2) (if you have rights) or [run here](https://rail.sandbox.ontotext.com/graphs-visualizations?config=8839d5b2864e47a4ad26d74f1f0423a2).

First we put this CONSTRUCT query in "Starting point> Start with graph query results":
```sparql
PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
PREFIX : <http://data.europa.eu/949/>

construct {?op1 :connectedTo ?op2}
where {
    ?sol a :SectionOfLine; :opStart ?op1c; :opEnd ?op2c; :canonicalURI ?solc.
    ?op1 a :OperationalPoint ; :canonicalURI ?op1c ;
      :inCountry <http://publications.europa.eu/resource/authority/country/BGR>.
    ?op2 a :OperationalPoint ; :canonicalURI ?op2c ;
      :inCountry <http://publications.europa.eu/resource/authority/country/BGR>.
}
```
- The query looks for Sections of Line (`sol`) that connect Operational Points (`op1, op2`).
  There are two relations from `sol`: `:opStart, :opEnd`
- We limit to only `op1, op2` in Bulgaria (`BGR`)
- Bulgaria doesn't use historical records, so we don't need to do distinct by `:canonicalURI`.
  Some other countries use them (see [Canonical URIs](#canonical-uris) and [Data Duplication](#data-duplication)) 
  so you'd need to complicate the query to eliminate duplicate `sol, op1, op2` that relate to the same canonicals.
  You can use `group by ?solc ?op1c ?op2c` and `sample` to select only one representative from each group.
- We won't visualize the `sol` as a node (only `op1, op2`) 
  so we return a synthetic (made-up) relation `connectedTo`

Then we put this SELECT query in "Node basics":
```sparql
PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
PREFIX : <http://data.europa.eu/949/>
select * {
    ?node :opType/skos:prefLabel ?type
    filter (lang(?type)="en")
    ?node :opName ?name; :uopid ?id
    bind(concat(?name,"\n",?type,": ",?id) as ?label)
}
```
It fetches two pieces of info per `?node`:
- `?type` which is used to color the nodes. 
  - By default `rdf:type` is used, but all our nodes have the same type `:OperationalPoint` so they all come out the same color.
  - By using the "business type" `:opType/skos:prefLabel` we get a meaningful type (as string) and different colors.
- `?label` which is displayed on the node. 
  - By default it uses `rdfs:label`, which works ok
  - But we do better by concatenating `?name, ?type, ?id` with proper separators.
    Workbench displays Up to 2 lines, so we put the name (which can be longer) on the first line, and the other two fields on the second line

After you run the visualization, in Settings increase "Maximum links to show" to 200.
I have [saved a graph view](https://rail.sandbox.ontotext.com/graphs-visualizations?saved=34f6c7d165e44ec38ddbf4f2b2c55a62) for you to try. If you improve the display by dragging nodes, you can save the results for others to see:

![](vizGraph-Bulgaria.png)

I've left the "Node expansion" query to its default, which selects all neighbors when you click on a node.
The result is not very good since it shows all kinds of stuff, that the info displayed in these extra nodes is not optimized.
I leave it as an exercise for the reader to figure out some better graph expansion functionality:

![](vizGraph-Kaspichan.png)

## Data Stories

Try to recreate some of the [RINF data stories](https://data-interop.era.europa.eu/data-stories) (competency questions) for ERA KG, see [source](https://github.com/Interoperable-data/ERA_vocabulary/tree/main/queries).

I leave this as an exercise for the reader.
