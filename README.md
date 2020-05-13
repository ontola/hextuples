# HexTuples

_This document serves as a work in progress / draft specification_

HexTuples is an [NDJSON](http://ndjson.org/) (Newline Delimited JSON) based RDF serialization format.
It is desgined to achieve the best possible performance in a JS context (i.e. the browser).

## Serializing

- A valid HexTuples document MUST be serialized using [NDJSON](http://ndjson.org/)
- Each array MUST consist of six strings.
- Each array represents one RDF statement / quad / triple
- The six strings in each array respectively represent  `subject`, `predicate`, `object`, `datatype`, `lang` and `graph`.
- When a field has no value, use an empty string: `""`

## Example

Turtle / N-Triples:

```n-triples
<https://www.w3.org/People/Berners-Lee/> <http://schema.org/birthDate> "1955-06-08"^^<http://www.w3.org/2001/XMLSchema#date>.
<https://www.w3.org/People/Berners-Lee/> <http://schema.org/birthPlace> <http://dbpedia.org/resource/London>.
```

Expresed in HexTuples:

```ndjson
["https://www.w3.org/People/Berners-Lee/", "http://schema.org/birthDate", "1955-06-08", "http://www.w3.org/2001/XMLSchema#date", "", ""]
["https://www.w3.org/People/Berners-Lee/", "http://schema.org/birthPlace", "http://dbpedia.org/resource/London", "http://www.w3.org/1999/02/22-rdf-syntax-ns#namedNode", "", ""]
```

## Motvation

HexTuples is designed by [Thom van Kalkeren](https://github.com/fletcher91/) (a colleague of mine) because he noticed that parsing / serialization was unnecessarily costly in our stack, even when using the relatively performant `n-quads` format.
Since HexTuples is serialized in NDJSON, it benefits from the [highly optimised JSON parsers in browsers](https://v8.dev/blog/cost-of-javascript-2019#json).
It uses NDJSON instead of regular JSON because it makes it easier to parse concatenated responses (multiple root objects in one document).
As an added plus, this enables streaming parsing as well, which gives it another performance boost.
Our JS RDF libraries ([link-lib](https://github.com/fletcher91/link-lib/), [link-redux](https://github.com/fletcher91/link-redux/)) have an internal RDF graph model which uses these arrays as well, which means that there is minimal mapping cost when parsing Hex-Tuple statements.
This format is especially suitable for real front-end applications that use dynamic RDF data.
