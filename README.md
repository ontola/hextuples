# HexTuples

_This document serves as a work in progress / draft specification_

HexTuples is an [NDJSON](http://ndjson.org/) (Newline Delimited JSON) based RDF serialization format.
It is desgined to support streaming parsing and provide great performance in a JS context (i.e. the browser).

## Serializing

- A valid HexTuples document MUST be serialized using [NDJSON](http://ndjson.org/)
- Each array MUST consist of six strings.
- Each array represents one RDF statement / quad / triple
- The six strings in each array respectively represent  `subject`, `predicate`, `object`, `datatype`, `lang` and `graph`.
- The `datatype` and `lang` fields are only used when the `object` represents a Literal value (i.e. not a URI, but a string / date / something else).
- When expressing an Object that is a NamedNode, use this string as the datatype: "http://www.w3.org/1999/02/22-rdf-syntax-ns#namedNode" ([discussion](https://github.com/ontola/hextuples/issues/1))
- When expressing an Object that is a BlankNode, use this string as the datatype: "http://www.w3.org/1999/02/22-rdf-syntax-ns#blankNode"
- The `graph` field does not support blank nodes. ([discussion](https://github.com/ontola/hextuples/issues/2))
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

## Motivation

HexTuples is designed by [Thom van Kalkeren](https://github.com/fletcher91/) (CTO of Ontola) because he noticed that parsing / serialization was unnecessarily costly in our full-RDF stack, even when using the relatively performant `n-quads` format.

- Since HexTuples is serialized in NDJSON, it benefits from the [highly optimised JSON parsers in browsers](https://v8.dev/blog/cost-of-javascript-2019#json).
- It uses NDJSON instead of regular JSON because it makes it easier to parse **concatenated responses** (multiple root objects in one document).
- NDJSON enables **streaming parsing** as well, which gives it another performance boost.
- Some JS RDF libraries ([link-lib](https://github.com/fletcher91/link-lib/), [link-redux](https://github.com/fletcher91/link-redux/)) have an internal RDF graph model which uses these HexTuples arrays as well, which means that there is minimal mapping cost when parsing Hex-Tuple statements.
This format is especially suitable for real front-end applications that use dynamic RDF data.

## Parsing HexTuples

We'll open source a proper parser soon, but this Typescript code should give you some idea of how to write a parser for HexTuples.

```ts
const object = (value: string, datatype: string, language: string): SomeTerm => {
  if (language) {
    return literal(value, language);
  } else if (datatype === 'http://www.w3.org/1999/02/22-rdf-syntax-ns#namedNode') {
    return namedNode(value);
  } else if (datatype === 'http://www.w3.org/1999/02/22-rdf-syntax-ns#blankNode') {
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
