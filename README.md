
[![stability-experimental](https://img.shields.io/badge/stability-experimental-orange.svg)](https://github.com/joethorley/stability-badges#experimental)
[![Travis-CI Build
Status](https://travis-ci.org/cboettig/emld.svg?branch=master)](https://travis-ci.org/cboettig/emld)
[![Coverage
Status](https://img.shields.io/codecov/c/github/cboettig/emld/master.svg)](https://codecov.io/github/cboettig/emld?branch=master)
[![CRAN\_Status\_Badge](http://www.r-pkg.org/badges/version/emld)](https://cran.r-project.org/package=emld)

<!-- README.md is generated from README.Rmd. Please edit that file -->

# emld

The goal of emld is to provide a way to work with EML metadata in the
JSON-LD format. At it’s heart, the package is simply a way to translate
an EML XML document into JSON-LD and be able to reverse this so that any
semantically equivalent JSON-LD file can be serialized into EML-schema
valid XML. The package has only three core functions:

  - `as_emld()` Convert EML from `xml` (or `json`) into a native R
    object, `emld` (list / S3 class).  
  - `as_xml()` Convert the native R format, `emld`, back into XML-schema
    valid EML.
  - `as_json()` Convert the native R format, `emld`, into `json`(LD).

This is very much **work in progress** The outline below illustrates
things we can do or will be able to do with this package. The examples
below are just a sketch of ideas so far, I hope to replace these with
richer examples that will probably be developed more fully as vignettes.

## Installation

You can install emld from github with:

``` r
# install.packages("devtools")
devtools::install_github("cboettig/emld")
```

## Motivation

In contrast to the existing [EML
package](https://ropensci.github.io/EML), this package aims to a very
light-weight implementation that seeks to provide both an intuitive data
format and make maximum use of existing technology to work with that
format. In particular, this package emphasizes tools for working with
linked data through the JSON-LD format.

Note that the JSON-LD format is considerably less rigid than the EML
schema. This means that there are many valid, semantically equivalent
representations on the JSON-LD side that must all map into the same or
nearly the same XML format. At the extreme end, the JSON-LD format can
be serialized into RDF, where everything is flat set of triples
(e.g. essentially a tabular representation), which we can query
directly with semantic tools like SPARQL, and also automatically coerce
back into the rigid nesting and ordering structure required by EML. This
ability to “flatten” EML files can be particularly convenient for
applications consuming and parsing large numbers of EML files. This
package may also make it easier for other developers to build on the
EML, since the S3/list and JSON formats used here have proven more
appealing to many R developers than S4 and XML serializations.

JSON-LD also makes it easier to extend EML with other existing semantic
vocabularies. The standard JSON-LD operations (e.g. framing, compaction)
make it easy for developers to specify desired data structures, filter
unnecessary terms and provide defaults for needed ones, or even define
custom property names, rather than working with the often cumbersome
prefixes or URIs of linked data.

``` r
library(emld)
library(jsonlite)
library(magrittr)
```

## Reading EML

The `EML` package can get particularly cumbersome when it comes to
extracting and manipulating existing metadata in highly nested EML
files. The `emld` approach can leverage a rich array of tools for
reading, extracting, and manipulating existing EML files.

### Parse & serialize

We can parse a simple example and manipulate is as a familiar list
object (S3 object):

``` r
f <- system.file("extdata/example.xml", package="emld")
eml <- as_emld(f)
eml$dataset$title
#> [1] "Data from Cedar Creek LTER on productivity and species richness\n  for use in a workshop titled \"An Analysis of the Relationship between\n  Productivity and Diversity using Experimental Results from the Long-Term\n  Ecological Research Network\" held at NCEAS in September 1996."
```

``` r
eml$dataset$title <- "A new title"
as_xml(eml, "test.xml")
```

We can prove that writing the list back into XML still creates a valid
EML file.

``` r
eml_validate("test.xml")
#> [1] TRUE
#> attr(,"errors")
#> character(0)
```

### Query

We can query it with SPARQL, a rich, semantic way to extract data from
one or many EML files. (The [W3C standard for
SPARQL-RDF](https://www.w3.org/TR/rdf-sparql-query/) provides a passable
introduction to the query syntax; haven’t found a more readable
alternative that is still reasonably feature-complete).

``` r
library(rdflib)
```

FIXME replace with an example(s) that makes better use of semantic
relationships.

``` r
f <- system.file("extdata/hf205.xml", package="emld")

as_emld(f) %>%
as_json("hf205.json")

sparql <- 
  'PREFIX eml: <http://ecoinformatics.org/>

  SELECT ?genus ?species ?northLat ?southLat ?eastLong ?westLong 

  WHERE { 
    ?y eml:taxonRankName "genus" .
    ?y eml:taxonRankValue ?genus .
    ?y eml:taxonomicClassification ?s .
    ?s eml:taxonRankName "species" .
    ?s eml:taxonRankValue ?species .
    ?x eml:northBoundingCoordinate ?northLat .
    ?x eml:southBoundingCoordinate ?southLat .
    ?x eml:eastBoundingCoordinate ?eastLong .
    ?x eml:westBoundingCoordinate ?westLong .
  }
'
  
rdf <- rdf_parse("hf205.json", "jsonld")
df <- rdf_query(rdf, sparql)
df
#> # A tibble: 0 x 0
```

We can also create queries with JQ, a [simple and powerful query
language](https://stedolan.github.io/jq/manual/) that also gives us a
lot of flexibility over the return structure of our results. JQ syntax
is both intuitive and well documented, and quite a bit easier than the
popular iterative strategies for rectangling JSON/list data using
`purrr`. Here’s an example query:

``` r
library(jqr)

as_emld(f) %>%
  as_json() %>% 
  as.character() %>%
  jq('.dataset.coverage.geographicCoverage.boundingCoordinates | 
       { northLat: .northBoundingCoordinate, 
         southLat: .southBoundingCoordinate }')
#> {
#>     "northLat": "+42.55",
#>     "southLat": "+42.42"
#> }
```

Some prototype examples of how we can use this to translate between EML
and <http://schema.org/Dataset> representations of the same metadata can
be found in
<https://github.com/cboettig/emld/blob/master/notebook/jq_maps.md>

### Flatten and Compact.

We can flatten it, so we don’t have to do quite so much sub-setting.
When we’re done editing, we can compact it back into valid EML.

``` r
library(jsonld)
flat <- as_emld(f) %>%
  as_json() %>% 
  jsonld_flatten('{"@vocab": "http://ecoinformatics.org/"}') %>%
  fromJSON(simplifyVector = FALSE)
flat <- flat[["@graph"]]
```

FIXME this would be way more useful if nodes all had `@type` and were
named by that type. Then we could do `flat$boundingCoordinates`.
Currently flattened objects are unnamed and untyped, so this is less
useful.

## Writing EML

This section is even more experimental currently, and may not be a good
direction for development. Nevertheless, it can illustrate some of the
convenience (and risk) of a simple S3 class.

Remember that `emld` objects are just nested lists, so you can create
EML just by writing lists:

``` r

me <- list(individualName = list(givenName = "Carl", surName = "Boettiger"))

eml <- list(dataset = list(
              title = "dataset title",
              contact = me,
              creator = me),
              system = "doi",
              packageId = "10.xxx")

as_xml(eml, "ex.xml")
testthat::expect_true(eml_validate("ex.xml") )
```

Note that we don’t have to worry about the order of the elements here,
`as_xml` will re-order if necessary to validate. (For instance, in valid
EML the `creator` becomes listed before `contact`.)

Clearly most users are unlikely to remember what the possible properties
are for each element, or which ones are attributes (which currently need
the `#` prefix). The `template()` function offers a quick way to address
this issue:

``` r
template("creator")
#> individualName: {}
#> organizationName: ~
#> positionName: ~
#> address: {}
#> phone: ~
#> electronicMailAddress: ~
#> onlineUrl: ~
#> userId: ~
#> id: ~
#> system: ~
#> scope: ~
```

returns a simple list object, cast as an `emld` S3 class so that it
prints cleanly. Note that properties which take other objects are
indicated by `{}` while properties taking text values are indicated by
`~`. If we assignt it to a variable instead of merely printing the
output of `template()`, we can then fill out values as needed (making
use of tab completion):

``` r
scott <- template("creator")
scott$individualName <- template("individualName")
scott$individualName$givenName <- list("Scott", "A")
scott$individualName$surName <- "Chamberlain"
scott$electronicMailAddress <- "scott@ropensci.org"
```

We can add this element to the EML list we already created:

``` r
eml$dataset$creator <- list(me, scott)
```

and confirm the file validates with this additional creator:

``` r
as_xml(eml, "ex.xml")
testthat::expect_true(eml_validate("ex.xml") )
```

It remains to be seen if this proves substantially easier than EML
creation with the S4-based constructors. Some things are certainly more
convienent: no `new`, no `ListOf` types, no remembering to index
repeated elements, no `.Data` slots, etc. On the other hand, S4 system
prevents assigning to a slot an object that is of the wrong type. The
list approach does not have this safeguard.

Ultimately it might be more useful as a developer tool to construct
helper functions that can streamline creation of EML.

### Going deeper

Several basic steps still to do: in particular, haven’t yet expanded out
`references` or EML’s semantic annotations into more native JSON-LD (and
vice-versa back into EML). The latter in particular might be a
convenient way to serialize additional semantic annotations onto an EML
description.
