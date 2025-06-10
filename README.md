# RailDataForum2025 SPARQL Tutorial

## Intro

[RailDataForum2025](https://www.era.europa.eu/agenda/agenda-rail-data-forum-2025) offers 3 "SPARQL for Beginners" sessions:
- June 11 9:30-10:30 and June 12 13:30-15:15 (a longer session of 1:45h).
- This will be led by Vladimir Alexiev, chief data architect of Graphwise/Ontotext.
- I assume some basic knowledge of the RDF graph model and the Turtle format (serialization),
  since SPARQL patterns are based on the same syntax.
- You can follow this by a "Practical Data Consumption" session that will be led by Ghislain Atemezing of ERA.

We will explain the basics of SPARQL, 
show some queries (competency questions), 
and an `ERAbot` chatbot that can generate SPARQL.

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
- [RINF data stories](https://data-interop.era.europa.eu/data-stories) (competency questions, see [source](https://github.com/Interoperable-data/ERA_vocabulary/tree/main/queries)): you can select and execute them directly
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
- Then use `Explain Resource` to see how the bot came up with an answer, and what query it used.
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

## Making SPARQL Queries

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
- `DISTINCT` eliminates duplicates from a result. You can use it in SELECT, or inside `group_concat()`
  Here we need it, but most of the times its use is a mistake, see next section
- We use `separator=";"` to more clearly see the different names, because one them includes a space
- We use `str(?label)` to convert the `langString` to a simple `string` 
  because otherwise the ordering will be first by lang tag, and then by value
- We use a subquery to enforce the ordering: simply using `order by` on the outer query doesn't work.
  - You may wonder how did I know that: I didn't! 
    I developed the query incrementally, which is a very common approach
  - Subqueries are wrapped in `{...}` and must select the variables to be exported outside of the subquery.
  - They are also used to enforce an order of execution, especially in federated queries

> An Rómáin; Errumania; Ir-Rumanija; Pyмъния; Roemenië; Romania; Romanya; Romanía; Romunija; Románia; România; Roménia; Roumanie; Rumania; Rumanía; Rumeenia; Rumenia; Rumunia; Rumunija; Rumunjska; Rumunsko; Rumänien; Rumænien; Rumānija; Rúmenía; Ρουμανία; Романија; Румунија; Румунія; Румыния; Румынія; رومانيا; ルーマニア; 罗马尼亚

