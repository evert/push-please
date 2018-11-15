---
date: 2018-11-15
title: "HTTP-client suggested Push Preference"
docname: draft-pot-prefer-push-00
category: std
author:
  -
    ins: Evert Pot
    email: me@evertpot.com
    uri: https://evertpot.com/
normative:
  RFC7540:
  RFC8288:
  RFC7240:
  I-D.ietf-httpbis-header-structure:
  W3C.REC-selectors-3-20181106:
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
  S-expression:
    -: "sexp"
    title: "S-expressions"
    target: https://en.wikipedia.org/wiki/S-expression

--- abstract

TODO

--- middle

# Introduction

HTTP/2 {{RFC7540}} allows a server to push requests and responses to a HTTP
client, in anticipation that the client will need them in the near future.

This mechanism is completely controlled by the server, and it is up to
implementors of services to anticipate what resources a client might need
next. Some implementations of this feature attempt to intelligently guess
which resources a client might need based on past behavior.

This specification defines a new HTTP header that allows a client to inform a
server of resources they will require next based on a link relation type
{{RFC8288}}.

# Rationale

Many HTTP-based services provide some mechanism to embed the HTTP response
bodies of resources into other HTTP resource. A common example of this when
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
2. Generating a set of resources can often be implemented on a server to
   be less time consuming that generating each response individually.

These mechanism also pose an issue. HTTP clients and intermediates and caches
are not aware of these embedded resources, because there was never a real HTTP
request.

By leveraging HTTP/2 push instead of poorly standardized embedding mechanisms,
it's possible for services to push subordinate resources as soon as possible,
generate HTTP responses as a "set" all while still taking advantage of existing
HTTP infrastructure.

In many REST apis, sub-ordiniate or embedded resources are identified by their
link relation. By using the link relation, it will be possible for a client
to indicate to a server which links they intent to follow, allowing a server
to only push the resources that the client knows it will need.

# The header format

**Note**: the following subsections contain several proposals. Only one should
stay in this specification, but all suggestions are currently listed for
completeness.

Each subsection contains an example of a `Prefer-Push` header, but also stated
as a new parameter of the `Prefer` header {{RFC7240}}. In this case also only 1
should stay in this document before duplication.

## Single-depth pushes

The simplest version of this only allows single depth push preferences. This
means that the `Prefer-Push` header can only be used for linked resources from
the context resource.

### Example using Prefer-Push header

This format should uses the "List" format from the Structured Headers format
{{I-D.ietf-httpbis-header-structure}}.

~~~~
GET /articles HTTP/1.1
Prefer-Push: item, author, "https://example.org/custom-rel"
~~~~

### Example using Prefer header

~~~~
GET /articles HTTP/1.1
Prefer: push="item, author, \"https://example.org/custom-rel\""
~~~~

## N-depth pushes

Hypermedia clients often will need to follow a chain of links, it might
therefore be benefitial to allow clients to specify relations at multiple
depths.

The following example requests for the server to push the resources linked
via the "item" and "icon" relation. For each of the resources linked via
the "item" relationship, it also requests the "author" relationship and
custom link relationship to be pushed.

### Example using Prefer-Push header

Because the Structered Headers format does not have a way to define headers
of arbitrarly depth, a custom format is used. This format is intended to
resemble existing HTTP headers but there's no direct equivalent.

~~~~
GET /articles HTTP/1.1
Prefer-Push: item(author, "https://example.org/custom-rel"), icon
~~~~

### Example using Prefer header

~~~~
GET /articles HTTP/1.1
Prefer: push="item(author, \"https://example.org/custom-rel\"), icon"
~~~~

## N-depth pushes with S-expression syntax

The following example expresses the same information, but instead of a custom
format it uses {{S-expression}}.

### Example using Prefer-Push header

~~~~
GET /articles HTTP/1.1
Prefer-Push: (item(author "https://example.org/custom-rel") icon)
~~~~

### Example using Prefer header

~~~~
GET /articles HTTP/1.1
Prefer: push="(item(author \"https://example.org/custom-rel\") icon)"
~~~~

## N-depth pushes with CSS syntax

A different suggestion was made using a combination of a subset of CSS3
selector syntax {{W3C.REC-selectors-3-20181106}} combined with the CSS
syntax of specifing urls.

### Example using Prefer-Push header

~~~~
GET /articles HTTP/1.1
Prefer-Push: item, item > author, item >
  url("https://example.org/custom-rel"), icon
~~~~

### Example using Prefer header

~~~~
GET /articles HTTP/1.1
Prefer: push="item, item > author, item >
  url(\"https://example.org/custom-rel\"), icon"
~~~~

## N-depth pushes with SparQL property paths

The following format is inspired by SparQL property paths.

### Example using Prefer-Push header

~~~~
GET /articles HTTP/1.1
Prefer-Push: item / ( author, <https://example.org/custom-rel> ),
  icon
~~~~

### Example using Prefer header

~~~~
GET /articles HTTP/1.1
Prefer: push="item / ( author, <https://example.org/custom-rel> ),
  icon"
~~~~

# Server pushes

When a server receives the `Prefer(-Push)` header, it can choose to push the
related resources.

It's possible for a resource to be references multiple times via different
link-relationships. The server must de-duplicate these responses.

A server is free to ignore push-requests.

# ABNF syntax

TODO when format is picked

# Using with "preload" relationship types

{{W3C.CR-preload-20171026}} defines a "preload" relationship type, that an
origin can use to inform a client to start fetching a resource, or a proxy
to initiate a HTTP/2 push.

Clients interacting with servers or proxies implementing "preload" could
discard `Prefer-Push: preload`, as it would be a no-op, but this is not
recommended as servers and proxies could still take this as a hint that
a Push is desired.

A distinct difference between `preload` and `Prefer-Push` is that `preload`
can be used by origin servers to inform clients and intermediates to fetch
and potentially push resources optimistically, but fundamentally `Prefer-Push`
is a completely client-driven mechanism.

# Security considerations

The Prefer-Push mechanism can potentially result in a large number of
resources being pushed. This can result in a Denial-of-Service attack.

A server must set reasonable restrictions around the amount of pushes it
sends. In the case of N-Depth pushes, servers SHOULD also set restrictions
around the depth it supports.

# IANA considerations

TODO: Put registry updates where a decision is made on Prefer/Prefer-Push.

# Acknowledgements

--- back

