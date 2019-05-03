---
date: 2018-12-02
title: "HTTP-client suggested Push Preference"
docname: draft-pot-prefer-push-01
category: std
author:
 -
    ins: E. Pot
    name: Evert Pot
    email: me@evertpot.com
    uri: https://evertpot.com/
normative:
  RFC7540:
  RFC8288:
  I-D.ietf-httpbis-header-structure:
  W3C.CR-preload-20171026:

informative:
  RFC4287:
  HAL:
    -: "hal"
    title: JSON Hypertext Application Language
    author:
      name: "Mike Kelly"
    date: 2012-06-07
    target: https://tools.ietf.org/html/draft-kelly-json-hal-00
  JSON-API:
    -: json-api
    title: JSON:API
    target: https://jsonapi.org/format/

--- abstract

`Prefer-Push` is a HTTP header that a client may use to request that a
server uses HTTP/2 Push to send related resources as identified by their
link relationships.

--- middle

# Introduction

HTTP/2 {{RFC7540}} allows a server to push request and response pairs to
HTTP clients. This can save round-trips between server and client and
reduces the total time required for a client to retrieve all requested
resources.

This mechanism is completely controlled by the server, and it is up to
implementors of services to anticipate what resources a client might need
next.

This specification defines a new HTTP header that allows a client to inform a
server of resources they will require next based on a link relation type
{{RFC8288}}.

# Rationale

Many HTTP-based services provide some mechanism to embed the HTTP response
bodies of resources into other HTTP resource. A common example of this is when
a resource is structured as a "collection of resources". Examples of this
include:

* The Atom Syndication Format {{RFC4287}} that encodes `ATOM:entry` XML
  elements for each subordinate.
* The [hal] format, which provides an `_embedded` element to embedding bodies
  of resources in other resources.
* The [json-api] format, which provides a `included` property to embed
  resources.

Embedding resource responses in other resources has two major peformance
advantages:

1. It reduces the number of roundtrips. A client can make a single HTTP request
   and get many responses.
2. Generating a related set of resources can often be implemented on a server
   to be less time consuming than generating each response individually.

These mechanisms also pose an issue. To HTTP clients and intermediaries such
as proxies and caches resources are opaque. They are not aware of a concept
of embedded resources.

One example where this might fail is if a client recieves a resource, embedded
in another resource, a cache might not be aware of this resource and serve a
stale, older version when this resource is requesed directly.

To keep the performance advantage of being able to generate a related set of
HTTP responses together, HTTP/2 Push could be an alternative to embedding.

HTTP/2 Push allows the server to initiate a request and response pair and send
them to the client early if the server thinks it will need them. Another
advantage of HTTP/2 push over embedding is that it allows resources of mixed
mediatypes to be pushed.

Servers can however not always anticipate which resources a client might want
pushed. To avoid guessing, this specification introduces a `Prefer-Push`
header that allows a client to inform a server which resources they will
need next.

In many REST apis, sub-ordiniate or embedded resources are identified by their
link relation. By using the link relation, it will be possible for a client
to indicate to a server which links they intent to follow, allowing a server
to only push the resources that the client knows it will need.

# The header format

This format should the "List" Data Type from the Structured Headers
specification {{I-D.ietf-httpbis-header-structure}}.

~~~~
GET /articles HTTP/1.1
Prefer-Push: item, author, "https://example.org/custom-rel"
~~~~

Each item in the list is a link relationship, as described in Web Links
{{RFC8288}}.

# Handling a Prefer-Push request

When a server receives the `Prefer-Push` header, it can choose to push the
related resources. It's up to the discretion of the implementor to decide
which resources to push. A server is also free to ignore push-requests.

If a server chooses to act on an item in the `Prefer-Push` list, the Link
Relationship should exist at the target resource. This specification does
not require that the link relationships get returned as HTTP `Link` headers.
The `Link` may be defined as `<link>` HTML element, or as a JSON property.
How the link is serialized is dependent on the media type.

# Using with "preload" relationship types

{{W3C.CR-preload-20171026}} defines a `preload` relationship type. This
relationship type can be used by an origin to inform a client or intermediate
to start fetching a resource, or a proxy to initiate a HTTP/2 push.

A distinct difference between `preload` and `Prefer-Push` is that `preload`
can be used by origin servers to inform clients and intermediates to fetch
and potentially push resources optimistically, but fundamentally `Prefer-Push`
is a completely client-driven mechanism.

These features can co-exist, but a wide adoption of client-driven suggestions
for pushes might eventually make `preload` unnecceary as in most cases clients
will have a better knowledge of the resources they need.

# Security considerations

The Prefer-Push mechanism can potentially result in a large number of
resources being pushed. This can result in a Denial-of-Service attack.

A server must set reasonable restrictions around the number of pushed
resources.

# IANA considerations

This document defines the `Prefer-Push` HTTP request fields and registers
them in the Permanent Message Header Fields registry.

## Prefer-Push {#prefer-push}
- Header field name: Prefer-Push
- Applicable protocol: HTTP
- Status: standard
- Author/Change controller: IETF
- Specification document(s): {{prefer-push}} of this document
- Related information: for Client Hints

# Acknowledgements

--- back

# Example

A server serves a document with a JSON-based media-type. The following example
document might represent a list of articles:

~~~~
HTTP/1.1 200 OK
Content-Type: application/vnd.example.links+json

{
   "links": [
      { "rel": "item", "href": "/article/1" },
      { "rel": "item", "href": "/article/2" },
      { "rel": "item", "href": "/article/3" },
      { "rel": "item", "href": "/article/4" },
      { "rel": "item", "href": "/article/5" }
   ]
   "total" : 5,
}
~~~~

A "Prefer-Push"-enabled client knows it will want to receive the full
representations of all articles. When the client receives the list of
articles via a "GET" request, it can indicate this preference with
the "Prefer-Push" header:

~~~~
GET /article HTTP/1.1
Accept: application/vnd.example.links+json
Prefer-Push: item
~~~~

Upon recieving this request, server may immediately generate the request
and response pairs for every "item" link in the collection and initiate
push streams for each.

# Changelog

## Changes since -00

* Added an abstract
* Updated rationale section significantly.
