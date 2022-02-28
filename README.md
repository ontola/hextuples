# HexTuples

_Status: draft_

_Version: 0.3.0_

HexTuples is a simple datamodel for dealing with linked data.
This document both describes the model and concepts of HexTuples, as well as the (at this moment only) serialization format: HexTuples-NDJSON.
It is very **easy to parse**, can be used for **streaming parsing** and is designed to be **highly performant** in JS contexts. 

## Concepts

### HexTuple

A single _HexTuple_ is an atomic piece of data, similar to an [RDF Triple](https://www.w3.org/TR/rdf-concepts/#section-triples) (also known as Statements or Quads).
A HexTuple cotains a small piece of information. 
HexTuples consist of six fields: `subject`, `predicate`, `value`, `datatype`, `language` and `graph`.

Let's encode the following sentence in HexTuples:

_Tim Berners-Lee, the director of W3C, is born in London on the 8th of June, 1955._

| Subject    | Predicate     | Value | DataType | Language | Graph |
|---------|----------------|------------|-----|-----|----|
| [Tim](https://www.w3.org/People/Berners-Lee/)     |[birthPlace](http://schema.org/birthPlace) | [London](http://dbpedia.org/resource/London)     | | |
| [Tim](https://www.w3.org/People/Berners-Lee/)     |[birthDate](http://schema.org/birthDate) | 1955-06-08     | [xsd:date](http://www.w3.org/2001/XMLSchema#date) | | 
| [Tim](https://www.w3.org/People/Berners-Lee/)     |[jobTitle](http://schema.org/jobTitle) | Director of W3C  | [xsd:string](http://www.w3.org/2001/XMLSchema#string) | en-US | 

### URI

URI stands for [Uniform Resource Identifier, specified in RDF 3986](https://tools.ietf.org/html/rfc3986).
The best known type of URI is the URL.
Although it is currently best practice to use mostly HTTPS URLs as URIs, HexTuples works with any type of URI.

### Subject

- The _subject_ is identifier of the thing the statement is about.
- This field is required.
- It MUST be a URI.

### Predicate

- The _predicate_ describes the abstract property of the statement.
- This field is required.
- It MUST be a URI.

### Value

- The _value_ contains the object of the HexTuple.
- This field is required.
- It can be any datatype, specified in the `datatype` of the HexTuple.

### Datatype

- The _datatype_ contains the object of the HexTuple.
- This field is optional.
- It MUST be a URI or an empty string.
- When the Datatype is a NamedNode, use: `globalId`
- When the Datatype is a BlankNode, use: `localId`

### Language

- The _datatype_ contains the object of the HexTuple.
- This field is optional.
- It MUST be an [RFC 3066 language tag](https://tools.ietf.org/html/rfc3066) or an empty string.

## Relation to RDF

The HexTuples datamodel closely resembles the RDF Data Model, which is the de-facto standard for linked data.
RDF statements are often called Triples, because they consist of a `subject`, `predicate` and `value`.
The `object` field is either a single URI (in Named Nodes), or a combination of three fields (in Literal): `value`, `datatype`, `language`.
This means that a single Triple can actually consist of _five_ fields: the `subject`, `predicate`, `value`, `datatype` and the `language`. 
A Quad statement also has a `graph`, which totals to six fields, hence the name: HexTuples.
Instead of making a distinction between Literal statements and NamedNode statements (which have two different models), HexTuples uses a single model that describes both.
**Having a single model for all statements (HexTuples), makes it easier to serialize, query and store data.**

## HexTuples-NDJSON

_This document serves as a work in progress / draft specification_

HexTuples-NDJSON is an [NDJSON](http://ndjson.org/) (Newline Delimited JSON) based HexTuples / RDF serialization format.
It is desgined to support streaming parsing and provide great performance in a JS context (i.e. the browser).

- A valid HexTuples document MUST be serialized using [NDJSON](http://ndjson.org/)
- HexTuples-NDJSON MIME type: `application/hex+x-ndjson; charset=utf-8`
- Each array MUST consist of six strings.
- Each array represents one RDF statement / quad / triple
- The six strings in each array respectively represent  `subject`, `predicate`, `value`, `datatype`, `lang` and `graph`.
- The `datatype` and `lang` fields are only used when the `value` represents a Literal value (i.e. not a URI, but a string / date / something else). In RDF, the combination of `value`, `datatype` and `lang` are known as `object`.
- When expressing an Object that is a NamedNode, use this string as the datatype: "globalId" ([discussion](https://github.com/ontola/hextuples/issues/1))
- When expressing an Object that is a BlankNode, use this string as the datatype: "localId"
- If the `graph` is a blank node (i.e. anonymous), use an underscore as the URI scheme: `_:myNode`. ([discussion](https://github.com/ontola/hextuples/issues/2)). Parsers SHOULD interpret these as blank graphs, but MAY discard these if they have no support for them.
- When a field has no value, use an empty string: `""`

### Example

English:

_Tim Berners-Lee was born in London, on the 8th of june in 1955._

Turtle / N-Triples:

```n-triples
<https://www.w3.org/People/Berners-Lee/> <http://schema.org/birthDate> "1955-06-08"^^<http://www.w3.org/2001/XMLSchema#date>.
<https://www.w3.org/People/Berners-Lee/> <http://schema.org/birthPlace> <http://dbpedia.org/resource/London>.
```

Expresed in HexTuples:

```ndjson
["https://www.w3.org/People/Berners-Lee/", "http://schema.org/birthDate", "1955-06-08", "http://www.w3.org/2001/XMLSchema#date", "", ""]
["https://www.w3.org/People/Berners-Lee/", "http://schema.org/birthPlace", "http://dbpedia.org/resource/London", "globalId", "", ""]
```

## Implementations

### Ontola TypeScript HexTuples Parser

* <https://github.com/ontola/hextuples-parser>

This Typescript code should give you some idea of how to write a parser for HexTuples.

```ts
const object = (value: string, datatype: string, language: string): SomeTerm => {
  if (language) {
    return literal(value, language);
  } else if (datatype === 'globalId') {
    return namedNode(value);
  } else if (datatype === 'localId') {
    return blankNode(value);
  }

  return literal(value, namedNode(datatype));
};

const lineToQuad = (h: string[]) => quad(
  h[0].startsWith('_:') ? blankNode(h[0]) : namedNode(h[0]),
  namedNode(h[1]),
  object(h[2], h[3], h[4]),
  h[5] ? namedNode(h[5]) : defaultGraph(),
);
```

### Python RDFlib

* <https://pypi.org/project/rdflib/>
* RDFLib is a pure Python package for working with RDF. 
* It supports parsing and serliazing RDF as HexTuples
* Internally (in Python objects), RDF parsed from HexTuples data is represented in a _Conjunctive Graph_, that is a multi-graph object
* HexTuples files must end in the file extension `.hext` for RDFlib to auto-recognise the format although files with any ending can be used if the format is given (`format=hext`)

An RDF format conversion tool using RDFLib that can convert from/to HexTuples is online at <http://rdftools.surroundaustralia.com/convert>.

## Motivation for HexTuples-NDJSON

HexTuples was designed by [Thom van Kalkeren](https://github.com/fletcher91/) (CTO of Ontola) because he noticed that parsing / serialization was unnecessarily costly in our [full-RDF stack](https://ontola.io/blog/full-stack-linked-data/), even when using the relatively performant `n-quads` format.

- Since HexTuples is serialized in NDJSON, it benefits from the [highly optimised JSON parsers in browsers](https://v8.dev/blog/cost-of-javascript-2019#json).
- It uses NDJSON instead of regular JSON because it makes it easier to parse **concatenated responses** (multiple root objects in one document).
- NDJSON enables **streaming parsing** as well, which gives it another performance boost.
- Some JS RDF libraries ([link-lib](https://github.com/fletcher91/link-lib/), [link-redux](https://github.com/fletcher91/link-redux/)) have an internal RDF graph model which uses these HexTuples arrays as well, which means that there is minimal mapping cost when parsing Hex-Tuple statements.
This format is especially suitable for real front-end applications that use dynamic RDF data.
