# Improving ERA SHACL Shapes and Data Flows

- Author: Vladimir Alexiev, chief data architect of Graphwise/Ontotext
- Date: 2025-08-20

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [Improving ERA SHACL Shapes and Data Flows](#improving-era-shacl-shapes-and-data-flows)
    - [Abstract](#abstract)
    - [Intro](#intro)
    - [ERA SHACL Stats](#era-shacl-stats)
    - [Minor Problems](#minor-problems)
    - [Issues with Regexps for Numbers](#issues-with-regexps-for-numbers)
    - [Use Complex SPARQLTarget but Simple SPARQLConstraint](#use-complex-sparqltarget-but-simple-sparqlconstraint)
    - [Constraints for SKOS Properties](#constraints-for-skos-properties)
    - [SKOS Schema But Not Exactly](#skos-schema-but-not-exactly)
    - [Incomplete SHACL](#incomplete-shacl)
    - [Closed Shapes](#closed-shapes)
    - [Ontology vs Shapes](#ontology-vs-shapes)
    - [Incremental Validation Scenarios](#incremental-validation-scenarios)
    - [References](#references)

<!-- markdown-toc end -->

## Abstract

The ERA KG is the result of a large-scale collaboration between Europe's rail entities
and is the foundation of EU rail data interoperability.
It is complex and relatively big, and SHACL data validation is used to improve data quality.

In this blog we describe Graphwise experience with complex SHACL shapes,
show how ERA SHACL shapes can be simplified by a third, 
how their performance can be optimized (instead of million times, run SPARQL once),
and discuss incremental validation scenarios that can further improve performance.

In particular, section [Constraints for SKOS Properties](#shacl-schemas-for-skos-properties) shows how to replace:
- 73 to 103 enum constraints that run SPARQL over 8M times (per enum prop instance)
- With 1 constraint that runs SPARQL once (then a few cheap SPARQLs per violation)

## Intro

The European Agency for Railways knowledge graph (ERA KG) is important for EU rail interoperability,
and is a success story in applying semantic technologies to industrial data.
The data comes from a number of parties: national infrastructure managers for the Register of Infrastructure (RINF),
authorization processes for the Type of Vehicles register (ERATV), etc.
Validating the data is an important part of ERA's data flow to improve data quality,
thus the [ERA ontology (Vocabulary)](https://gitlab.com/era-europa-eu/public/interoperable-data-programme/era-ontology/era-ontology) is complemented by a number of [SKOSConcept Schemes](https://gitlab.com/era-europa-eu/public/interoperable-data-programme/era-ontology/era-ontology/-/tree/main/era-skos) and [SHACL Shapes](https://gitlab.com/era-europa-eu/public/interoperable-data-programme/era-ontology/era-ontology/-/tree/main/era-shacl).

Graphwise has extensive experience with SHACL validation in complex projects, as can be seen from the following references.
- [1] describes advanced SHACL validation for an energy Knowledge Graph.
  - [2] uses a "literate programming style" to extract the shapes from the specification.
- [5] describes a number of suggested improvements to the Electrical CIM/CGMES SHACL
  (it is the result of a consulting contract with ENTSO-E).
  In addition to the `SPARQLTarget` improvement described below, it includes a number of practical considerations that may be useful for the ERA SHACL.

We have examined the ERA SHACL shapes, and make some suggestions for improvement based on this knowledge.
Most reference links focus on a way to improve the performance of SHACL constraints by
leaving the heavy lifting to `SPARQLTarget` (a query that is run few times)
and simplifying `SPARQLConstraint` (that is run very many times).

[6] describes a SHACL performance and conformance benchmark based on ERA SHACL shapes
that has tested the following engines:  `dotnet_rdf, jena, maplib, pyshacl, rdf4j, topbraid, corese, rdfunit`.
- [7] describes some problems with the setup of rdf4j.

## ERA SHACL Stats
[6] lists the following statistics; the paper is based on version 3.0.1 of the ERA KG, so we assume that the same version of ERA SHACL was used:
19 main shapes (`NodeShape`), 196 property shapes (`PropertyShape`).
275 constraints: 215 SHACL-core and 60 SHACL-SPARQL (`SPARQLConstraint`) constraints.


We counted [SHACL Shapes](https://gitlab.com/era-europa-eu/public/interoperable-data-programme/era-ontology/era-ontology/-/tree/main/era-shacl) in the latest version 3.1.1, and here are some statistics:
- 56 main shapes (`NodeShape`), all are targeted by class (`targetClass`).
- 305 property shapes (`PropertyShape`).
- 177 SHACL-SPARQL (`SPARQLConstraint`) shapes that are used 298 times (`sh:sparql`).
  For example, each SHACL-SPARQL constraint in `RINF-tunnels.ttl` is used twice, eg:
```ttl
era-sh:TunnelShape                      sh:sparql era-sh:CrossSectionAreaApplicability .
era-sh:CommonCharacteristicsSubsetShape sh:sparql era-sh:CrossSectionAreaApplicability .
```

In the remainder, we make some analysis of ERA SHACL shapes, and some suggestions for improvement.

## Minor Problems

Consider the following shape from `RINF-sol-tracks.ttl`:

```ttl
era-sh:SectionOfLineShape a sh:NodeShape ;
  sh:targetClass era:SectionOfLine .
  sh:sparql era-sh:NoRepeatedTrackIdsSoL.

era-sh:NoRepeatedTrackIdsSoL a sh:SPARQLConstraint ;
  era:affectedProperty era:trackId;
  era:affectedClass  era:SectionOfLine;
  era:affectedClass  era:RunningTrack;
  era:scope "global";
  rdfs:comment "Each track shall have unique identification or number within the SoL. This number cannot be used for naming any other track in the same SoL."@en ;
  era:rinfIndex "1.1.1.0.0.1" ;
  sh:severity sh:Violation ;
  sh:message "trackId (1.1.1.0.0.1): Each track shall have unique identification or number within the SoL. This number cannot be used for naming any other track in the same SoL. There is a problem with SoL {$this} ({?solLabel}) and tracks {?track1} ({?track1Label}) and {?track2} ({?track2Label}), since they have the same identifier: {?value}."@en ;
  sh:prefixes era:;
  sh:select """
    PREFIX era: <http://data.europa.eu/949/>
    PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
    SELECT $this ?solLabel ?track1 ?track1Label ?track2 ?track2Label ?value
      WHERE {
        $this era:hasPart ?track1 .
        $this era:hasPart ?track2 .
        ?track1 era:trackId ?value .
        ?track2 era:trackId ?value .
        FILTER(?track1 != ?track2) .
        OPTIONAL{$this rdfs:label ?solLabel} .
        OPTIONAL{?track1 rdfs:label ?track1Label} .
        OPTIONAL{?track2 rdfs:label ?track2Label} .
      } """ .
```
It has a few problems:
- It will return each pair of violating tracks twice.
  It'd be better to use `FILTER(str(?track1) < str(?track2))` to return only one of these permutations.
- `sh:prefixes era:` should declare all needed prefixes,
  so individual queries don't need to use any `PREFIX` declarations.
  Eg issue [ERA-SHACL-Benchmark issue 1](https://github.com/alexisimo/ERA-SHACL-Benchmark/issues/1) (duplicate prefixes)
  would not have occurred if this guideline was followed.

A typographic issue:
- Shapes (including SPARQL queries of which we have to say more below) use a mix of spaces and tab chars. 
  These should be replaced by spaces only, with proper identation of Turtle and the query.

## Issues with Regexps for Numbers

In a prior version ERA SHACL had some unanchored regexps (`sh:pattern`), i.e. checked only the middle of a value.

This has been fixed, but a few problems remain (the same appears in `era-sh:VNvsupovtrp`):
```ttl
era-sh:VNvallowovtrp a sh:PropertyShape ;
  rdfs:comment "Speed limit allowing the driver to select the “override” function in km/h. "@en ;
  sh:datatype xsd:unsignedInt ;
  sh:minInclusive 0 ;
  sh:maxInclusive 600 ;
  sh:message "vNvallowovtrp (1.1.1.3.2.16.3, 1.2.1.1.1.16.3): The track or subset with common characteristics defines a speed limit allowing the driver to select the “override” function (V_NVALLOWOVTRP). This error is due to having more than one value, having a value that is not an integer or having an integer that does not follow the pattern [NNF], with N a decimal number (0÷9), F=(0|5), max. `600`."@en ;
  sh:path era:vNvallowovtrp ;
  sh:pattern "^([1-9]\\d{0,1}[0,5]|[0,5])$" ;
  sh:severity sh:Violation .
```
- The regex `[0,5]` is wrong because it allows comma. The correct one is `[05]`
- Given the `sh:minInclusive, sh:maxInclusive` range, 
  the whole `pattern` can be simplified to just this: `"[05]$"`,
  i.e. requiring the last digit to be 0 or 5.

Patterns are often over-complicated.
- They use backslash and sometimes groups unnecessarily. Eg this can be simplified:
```
era-sh:MaximumAltitude sh:pattern "^(\\+|\\-)\\d{1,4}$" ;
# simpler:
era-sh:MaximumAltitude sh:pattern "^[+-]\\d{1,4}$" ;
```

Apparently the XML specs are not clear enough about number of digits (eg whether `[NN]` means exactly 2 digits or at most 2 digits).
For example the following requirement
> gradient profile value that must be a string with a sequence of values in the format `[+/-][NN.N]([+/-][NNNN.NNN])`

is translated to this regex:
```ttl
  	sh:pattern "^((\\+|\\-)([1-9]\\d{1}|[0-9]|00)\\.\\d{1,3}(\\((\\+|\\-)?([1-9]\\d{1,2}|[0-9])\\.\\d{1,3}\\))?)(,(\\+|\\-)([1-9]\\d{1}|[0-9])\\.\\d{1,3}(\\((\\+|\\-)?([1-9]\\d{1,2}|[0-9])\\.\\d{1,3}\\))?)*$" ;
```
Not only it is incredibly ugly, I also think it's incorrect:
- The part `([1-9]\\d{1}|[0-9]|00)` allows 2-digit numbers, but 0..9 must be spelled with 1-digit: this is inconsistent
- The part `\\d{1,3}` allows 1..3 digits in the fractional part. But the requirement is for 1 fractional digit

Requiring particular lexical representation of numbers (eg no leading zeros or presence of leading `+` sign) 
is a leftover from the XML schemas and is IMHO unnecessary in RDF.

It is better to check numbers for range rather than using patterns. Eg the above `MaximumAltitude` shape can be changed to:
```
era-sh:MaximumAltitude sh:minInclusive -9999; sh:maxInclusive +9999;
```

I think that there is an error below: 
the regex allows 1..3 digit numbers, but numbers greater than 330 require at least 3 digits to write.
```
era-sh:MinimumWheelDiameter
	sh:path era:minimumWheelDiameter ;
	sh:minInclusive 330 ;
	sh:pattern "^([1-9]\\d{1,2}|[0-9])$" ;
```
Perhaps `sh:maxInclusive 330` was meant, 
and the shape author got confused because the prop name includes "minimum".
If you focus on numeric ranges rather than regexp, such errors an be avoided.

## Use Complex SPARQLTarget but Simple SPARQLConstraint

The SPARQLConstraint in the previous section has a much bigger problem:
this complex SPARQL will run hundreds of thousands of times, **once per target instance** (`era:SectionOfLine` and `era:RunningTrack`).

A well-known trick is to use `sh:SPARQLTarget` **once** to do the heaviy lifting and find all violations,
after which a `sh:SPARQLTarget` will run a few times (**once per violation**)
and merely populate the `sh:ViolationReport` parameters.
See the References for Graphwise's experience with this trick since at least 2022.

We could rewrite the constraint as follows:

```sparql
era-sh:NoRepeatedTrackIdsSoL a sh:NodeShape;
  era:affectedProperty era:trackId;
  era:affectedClass era:SectionOfLine;
  era:affectedClass era:RunningTrack;
  era:scope "global";
  rdfs:comment "Each track shall have unique identification or number within the SoL. This number cannot be used for naming any other track in the same SoL."@en ;
  era:rinfIndex "1.1.1.0.0.1" ;
  sh:severity sh:Violation ;
  sh:message "trackId (1.1.1.0.0.1): Each track shall have unique identification or number within the SoL. This number cannot be used for naming any other track in the same SoL. There is a problem with SoL {$this} ({?solLabel}) and tracks {?track1} ({?track1Label}) and {?track2} ({?track2Label}), since they have the same identifier: {?value}."@en ;
  sh:target [a sh:SPARQLTarget;
    sh:prefixes era:;
    sh:select """
      SELECT distinct $this {
        VALUES ?class {era:SectionOfLine era:RunningTrack}
        $this a ?class;
        $this era:hasPart ?track1 .
        $this era:hasPart ?track2 .
        ?track1 era:trackId ?value .
        ?track2 era:trackId ?value .
        FILTER(str(?track1) < str(?track2))}"""];
  sh:sparql [a sh:SPARQLConstraint;
    sh:prefixes era:;
    sh:select """
      SELECT $this ?solLabel ?track1 ?track1Label ?track2 ?track2Label ?value {
        $this era:hasPart ?track1 .
        $this era:hasPart ?track2 .
        ?track1 era:trackId ?value .
        ?track2 era:trackId ?value .
        FILTER(str(?track1) < str(?track2))
        OPTIONAL{$this rdfs:label ?solLabel} .
        OPTIONAL{?track1 rdfs:label ?track1Label} .
        OPTIONAL{?track2 rdfs:label ?track2Label}}"""] .
```

Notes:

- You may notice we used `distinct` in the target query but no `distinct` in the constraint query.
  The reason is that the same `$this` may have several track pairs in violation, and we want to report all of them.
  While `distinct` is an expensive operation, the set of violating nodes is expected to be small, so it's ok to use it.
- Depending on how the validation engine populates `sh:message`,
  it may need to be moved into the `SPARQLConstraint`
- Nominally, [SPARQLTarget](https://www.w3.org/TR/shacl-af/#h-sparqltarget) is part of SHACL Advanced while [SPARQLConstraint](https://www.w3.org/TR/shacl/#example-sparql-constraint) is part of SHACL basic.
  However, all validation engines that implement the one SPARQL feature are also likely to implement the other,
  so there is little harm in using SPARQL targeting.
  - For example, out of the engines tested by [6]
    `jena, maplib, pyshacl, rdf4j, topbraid` support `SPARQLTarget` [9],
    `corese, rdfunit` are no longer being maintained,
    and I've posted an issue [10] for `dotnet_rdf` to implement this feature.

## Constraints for SKOS Properties

ERA uses 67 thesauri `skos:ConceptScheme`.
There are 103 enumeration properties with `rdfs:range skos:Concept`,
and each uses the `era:inSkosConceptScheme` annotation property
to specify which is the valid scheme for those concepts.

73 SPARQL constraints are dedicated to checking the conformance of values to the prescribed SKOS schemes.
They are distributed in the SHACL files as follows:
```
grep -c :inSkosConceptScheme *.ttl | grep -v :0
RINF-basic-net-reference.ttl:2
RINF-contact-line-systems.ttl:2
RINF-etcs.ttl:6
RINF-net-element.ttl:1
RINF-operational-points.ttl:1
RINF-organisation-role.ttl:1
RINF-parameter-applicability.ttl:1
RINF-platforms.ttl:2
RINF-sections-of-line.ttl:1
RINF-sidings.ttl:1
RINF-signal.ttl:2
RINF-sol-op-tracks.ttl:28
RINF-sol-tracks.ttl:18
RINF-subsidiary-location.ttl:1
RINF-train-detection-systems.ttl:5
RINF-tunnels.ttl:1
```

I am not sure why there are only 73.
Some of the constraints are used in several NodeShapes, eg `era-sh:LoadCapabilityLineCategorySKOS`
is used for both `era-sh:CommonCharacteristicsSubsetShape` and `era-sh:RunningTrackShape`.
But there should have been a SPARQL constraint for each enumeration property, i.e. 103 such constraints.

Each constraint works on a unique enumerated property, eg:
```ttl
era-sh:RunningTrackShape
  a sh:NodeShape ;
  sh:targetClass era:RunningTrack;
  sh:sparql era-sh:LoadCapabilityLineCategorySKOS.

era-sh:CommonCharacteristicsSubsetShape
  a sh:NodeShape ;
  sh:targetClass era:CommonCharacteristicsSubset ;
  sh:sparql era-sh:LoadCapabilityLineCategorySKOS.

era-sh:LoadCapabilityLineCategorySKOS
  a sh:SPARQLConstraint ;
  era:affectedProperty era:loadCapabilityLineCategory;
  era:affectedClass  era:LoadCapability;
  era:scope "local";
  rdfs:comment "Load capability is a combination of the line category and speed at the weakest point of the track."@en;
  era:rinfIndex "1.1.1.1.2.4" ;
  sh:severity sh:Violation ;
  sh:message "Indication of the load capability line category (1.1.1.1.2.4):): The track or subset with common characteristics  {$this}  has a value {?concept} that is not one of the predefined values and cannot be converted into a SKOS concept on this list: http://data.europa.eu/949/concepts/load-capability-line-categories/LoadCapabilityLineCategories."@en ;
  sh:prefixes era:;
  sh:select """
    PREFIX era: <http://data.europa.eu/949/>
    PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
    PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
    SELECT $this ?concept
    WHERE {
        $this era:trackLoadCapability ?loadCapability .
        ?loadCapability era:loadCapabilityLineCategory ?concept .
        era:loadCapabilityLineCategory era:inSkosConceptScheme ?conceptScheme .
        FILTER NOT EXISTS {?concept skos:inScheme ?conceptScheme}} """ .
```
Complex queries similar to the above will run millions of times, per each ERA KG instance that has enumerated properties.
Considering that a class can have multiple enumerated properties, that makes **many millions** of times.
Indeed, the following query at https://rail.sandbox.ontotext.com/sparql (which has ERA KG of January 2025 loaded) returns 8,336,142 results:
```sparql
PREFIX era: <http://data.europa.eu/949/>
select * {
    ?this ?property ?value.
    ?property era:inSkosConceptScheme ?scheme
}
```

We can replace all these 73 or 103 constraints with **one** generalized constrant:
```ttl
era-sh:InSkosConceptScheme a sh:NodeShape;
  sh:severity sh:Violation ;
  sh:message "Instance {$this} of class {?class} has enumerated property {?property} with value {?value} that is not in the list {?scheme}."@en ;
  sh:target [a sh:SPARQLTarget;
    sh:prefixes era:;
    sh:select """
      SELECT distinct $this {
        $this ?property ?value .
        ?property era:inSkosConceptScheme ?scheme .
        FILTER NOT EXISTS {?value skos:inScheme ?scheme}}"""];
  sh:sparql [a sh:SPARQLConstraint;
    sh:prefixes era:;
    sh:select """
      SELECT $this ?class ?property ?value ?scheme {
        $this a ?class; ?property ?value .
        ?property era:inSkosConceptScheme ?scheme .
        FILTER NOT EXISTS {?value skos:inScheme ?scheme}}"""] .
```
Notes:
- This is a lot more efficient because the first (target) query will run only **once**.
  The second (constraint) query will run a few times (assuming there aren't too many violating nodes), and with prebound `$this` so it is inexpensive.
- It also reduces the number of ERA shapes **by about a third**, so is a huge win in maintainability.
- We use `distinct` in the first query but not the second, since we want to report all `?property value` pairs that are in violation.
- We omitted a lot of the metadata (`era:affectedProperty, era:affectedClass, era:rinfIndex`).
  These can be multivalued, so we can add them to the generalized shape.
  We'll only lose the correlation between the 3 metadata props, but such is present in the ontology.
  The message is generalized properly to be accurate.

## SKOS Schema But Not Exactly

16 of the enumeration properties have a strange variation, namely `REGEX(?concept, "/eratv/")`.
Here is the distribution per SHACL file:
```
grep -c /eratv/ *.ttl | grep -v :0
RINF-contact-line-systems.ttl:1
RINF-etcs.ttl:3
RINF-platforms.ttl:1
RINF-sol-op-tracks.ttl:7
RINF-sol-tracks.ttl:4
RINF-train-detection-systems.ttl:2
RINF-tunnels.ttl:1
```

Eg here is one of them:
```ttl
era-sh:TemperatureRangeSKOS
  sh:message "Indication of the temperature range (1.1.1.1.2.6):): The track {$this} has a value {?concept} that is not one of the predefined values and cannot be converted into a SKOS concept on this list: http://data.europa.eu/949/concepts/temperature-ranges/TemperatureRanges."@en ;
  sh:prefixes era:;
  sh:select """
    SELECT $this  ?concept
    WHERE {
        $this era:temperatureRange ?concept .
        era:temperatureRange era:inSkosConceptScheme ?conceptScheme .
		FILTER (NOT EXISTS {?concept skos:inScheme ?conceptScheme} || REGEX(?concept, "/eratv/"))}""" .
```

This intends to say:
> Normally `temperatureRange` should only have values from the `TemperatureRanges` concept scheme.
> However, a URL that has `/eratv/` anywhere in it is also fine.

In reality `REGEX(?concept)` will always return UNDEF because `?concept` is a URL not a string.
One needs to use `REGEX(STR(?concept))` for this to work as intended.
Reported as [era-ontology issue 118](https://gitlab.com/era-europa-eu/public/interoperable-data-programme/era-ontology/era-ontology/-/issues/118)

This seems to be a temporary fix until ERATV enumerations are cleaned up and converted to SKOS.
But `sh:message` conveniently omits this detail, therefore is inaccurate.

We can easily accommodate this special case:
- Add a new prop annotation attribute `era:anyEratvAllowed` (boolean flag) to designate such "open enumeration props":
````
era:temperatureRange era:inSkosConceptScheme <.../TemperatureRanges>; era:anyEratvAllowed true
````
- Split the constraint in two:
  - `era-sh:InSkosConceptScheme` checks that the flag is missing or false, then checks for the scheme strictly
  - `era-sh:InSkosConceptSchemeOrAnyEratv` checks that the flag is true, then checks for the scheme in the lax way (also allows `REGEX`)

## Incomplete SHACL

I asked myself whether the SHACL shapes are complete, i.e. do they check all statements.
One could write SPARQL queries to compare the coverage of SHACL shapes 
against instance data (i.e. instantiated props).
- Using the standard `sh:property/sh:path` presents some difficulty 
  since it could be a simple prop, or a blank node expressing a prop path;
  and SPARQL constraints hide the props in a SPARQL string.
- But ERA has custom annotation props `era:affectedClass, era:affectedProperty`
  that explicate the intent of each shape, so such check should be easy.

I was too lazy to do this for all props, but I got curious about two special props.
They are some of the most populated props (see [8] section "Property Count"):
- In the Jan 2025 ERA KG, `era:notApplicable` has 6.27M instances, `era:notYetAvailable` has 5.3M
- They are used to describe for a rail resource, which expected properties are not there, and why

This query:
```
PREFIX owl: <http://www.w3.org/2002/07/owl#>
PREFIX era: <http://data.europa.eu/949/>
select ?prop (count(*) as ?c) {
    ?this era:notApplicable|era:notAvailable ?prop
    filter (not exists {?prop a owl:DatatypeProperty}
         && not exists {?prop a owl:ObjectProperty})
} group by ?prop order by ?prop
```
finds `era:pantographHead`:
this prop is not defined by the ontology, yet is declared missing 20,403 times.

## Closed Shapes
Imagine some infrastructure manager misspelled the above prop as `era:pantografHead` in his data.
The prop is optional, so a SHACL shape wouldn't catch that the correctly spelled prop is missing.

Unless one uses `sh:closed true` to close off the shape and forbid random misspelled props.

## Ontology vs Shapes

The ERA ontology should describe all classes and properties, 
and ERA SHACL should validate all of them.

But as [8] sections "ERA Class Count" and "ERA Property Count" show,
ERA KG 3.0 uses some classes and props that are not declared in the ontology 
or lack the link `isDefinedBy` to the ontology.

And as we have shown above, some properties are not covered by shapes.
For example, none of the props below are mentioned in the ontology or the shapes:
```
grep "pantographHead|minimumVerticalRadius|notApplicable|notAvailable" ontology.ttl era-shacl/*.ttl
```
But they appear in ERA KG 3.0.

ERA has started on a continuous integration and delivery (CI/CD) process
that includes some sanity checks.
The checks should be expanded to cover the full coverage 
of terms appearing in the KG, by term definitions in the ontology and validations by shapes.

## Incremental Validation Scenarios

[6] focuses on in-memory validation scenarios that go like this:
run a command-line tool, give it a bunch of data and SHACL files, 
it loads them all in memory, then spits out `ValidationReports`.

The ERA team currently uses the same approach,
but it limits the scalability of the KG and requires a lot of memory for validation.

A large KG should be "live" in the sense that most of the KG stays "at rest",
while various smaller parts of it evolve at their own pace, without disturbing the parts at rest.
Such changing parts can be:
1. Full subgraph corresponding to a particular country, coming from the respective Infrastructure Manager
   (eg in addition to the full ERA KG, [6] measures performance on the ES and FR subsets)
2. Smaller subgraph of data of an Infrastructure Manager (eg updating info about one `Tunnel`)
3. Slow-changing shared data, eg ontologies and SKOS thesauri
4. Data migration activities that work across the KG, but change only a small number of properties/nodes

[5] section [In-memory vs On-disk Databases and Incremental Validation](https://github.com/Sveino/Inst4CIM-KG/tree/develop/shacl-improved#in-memory-vs-on-disk-databases-and-incremental-validation)
describes validation scenarios for the electrical CIM/CGMES
and empasizes the benefits of using:
- The database to do the validation.
- Using incremental validation, where only an incoming transaction is validated,
  but in conjunction with all data at rest.
- The benefits of using SHACL-core (i.e. standard constraint components),
  which are declarative and therefore the validation engine 
  can determine which data triple patterns are checked by which shapes,
  thus minimizing validation effort.
  In contrast, it's hard to determine whether a SPARQL constraint needs to be run or not,
  for the triple patterns that appear in a transaction.
  
Incremental validation is especially important for CIM/CGMES because it includes Difference Models,
and being able to assume that the base model of a difference model is valid,
allows very efficient validation.

But I believe incremental validation is also important for ERA.
The current command-line validation takes only data triples,
but cannot take a data transaction (set of deleted and inserted triples), so it's limited:

- Scenario 1 above can be validated incrementally:
  validate only the data from an individual Infrastructure Manager,
  complementing it with shared data (e.g. SKOS thesauri) and cross-border reference OPs.
- Scenario 2 has to be handled like scenario 1, 
  i.e. by validating even the unchanged data of an Infrastructure Manager.
- Scenarios 3 and 4 can only be handled by full revalidation.

## References

1. Vladimir Alexiev,
   [Advanced SHACL Data Validation for the Transparency Energy KG](https://docs.google.com/presentation/d/1Hhxmx2YDnaxlaU5KeafjRJSDlVgHRz1z/edit).
   Presentation at Ontotext Demo Days, May 2022.
   [video](https://youtu.be/4JGSui7Uq_Y)
2. Vladimir Alexiev, Viktor Ribchev, Miroslav Chervenski, Nikola Tulechki, Mihail Radkov, Antoniy Kunchev, Radostin Nanov.
   [Transparency EKG Requirements Specification, Architecture and Semantic Model](https://transparency.ontotext.com/spec/#eic-compatible-with-function), section 3.6.10 EIC-compatible-with-function.
   Ontotext, 28 Sep 2022
3. Ontotext GraphDB documentation. SHACL Validation, section SPARQL capabilities in SHACL shapes.
   [Version 10.3](https://graphdb.ontotext.com/documentation/10.3/shacl-validation.html#sparql-capabilities-in-shacl-shapes) of 17 July 2023.
   [Latest version 11.0](https://graphdb.ontotext.com/documentation/11.0/shacl-validation.html#sparql-capabilities-in-shacl-shapes) of 7 May 2025.
4. Radostin Nanov.
   [SHACL-ing the Data Quality Dragon II: Application, Application, Application!](https://www.ontotext.com/blog/shacl-ing-the-data-quality-dragon-ii-application-application-application/#content-4-2), section SHACL-SPARQL performance.
   Ontotext blog post, 17 Nov 2023.
5. Vladimir Alexiev. [CIM/CGMES SHACL Improvements](https://github.com/Sveino/Inst4CIM-KG/tree/develop/shacl-improved), sections
   [Use Complex SPARQLTarget but Simple SPARQLConstraint](https://github.com/Sveino/Inst4CIM-KG/tree/develop/shacl-improved#use-complex-sparqltarget-but-simple-sparqlconstraint) and
   [Don't Overuse SHACL SPARQL](https://github.com/Sveino/Inst4CIM-KG/tree/develop/shacl-improved#dont-overuse-shacl-sparql).
   Github project, Jan 2025.
6. Edgar A. Martínez Sarmiento, Edna Ruckhaus, Jhon Toledo, Daniel Doña and Oscar Corcho.
   ERA-SHACL-Benchmark: A real data SHACL performance and reporting quality benchmark for in-memory SHACL engines.
   ISWC 2025 Resource track (submitted), June 2025.
7. Havard Ottestad, Vladimir Alexiev.
   [Use correct rdf4j setup; ERA-SHACL-Benchmark#4](https://github.com/alexisimo/ERA-SHACL-Benchmark/issues/4).
   Github issue, June 2025.
8. Vladimir Alexiev. [RailDataForum2025 SPARQL Tutorial](https://github.com/VladimirAlexiev/RailDataForum2025-SPARQL#readme).
   Github project, June 2025.
9. Vladimir Alexiev. [Which engines support SPARQLTarget? ERA-SHACL-Benchmark#6](https://github.com/alexisimo/ERA-SHACL-Benchmark/issues/6).
   Github issue, June 2025.
10. Vladimir Alexiev. [SHACL: implement SHACLTarget dotnetrdf#726](https://github.com/dotnetrdf/dotnetrdf/issues/726)
    Github issue, June 2025.
