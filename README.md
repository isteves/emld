
[![stability-experimental](https://img.shields.io/badge/stability-experimental-orange.svg)](https://github.com/joethorley/stability-badges#experimental) [![Travis-CI Build Status](https://travis-ci.org/cboettig/emld.svg?branch=master)](https://travis-ci.org/cboettig/emld) [![Coverage Status](https://img.shields.io/codecov/c/github/cboettig/emld/master.svg)](https://codecov.io/github/cboettig/emld?branch=master)

<!-- README.md is generated from README.Rmd. Please edit that file -->
emld
====

The goal of emld is to provide a way to work with EML metadata in the JSON-LD format.

Installation
------------

You can install emld from github with:

``` r
# install.packages("devtools")
devtools::install_github("cboettig/emld")
```

**Work in progress** The outline below illustrates things we can do or will be able to do with this package. More examples forthcoming soon.

``` r
library(emld)
```

Reading EML
-----------

`emld` excels at reading, extracting, and manipulating existing EML files.

### Parse & serialize

We can parse a simple example and manipulate is as a familar list object (S3 object):

### Flatten and Compact.

We can flatten it, so we don't have to do quite so much subsetting. When we're done editing, we can compact it back into valid EML.

### Query

We can query it with SPARQL, a rich, semantic way to extract data from one or many EML files.

We can query it with JQ, a simple and powerful query language that gives us a lot of flexibility over the return structure of our results:

Writing EML
-----------

We can also create EML from scratch using lists with help from the `template` function.

Let's create a minimal EML document
