---
layout: post
title: "HTTP APIs :: Content Negotiation"
tag: "blog"
---

It doesn't usually take too long, once clients start using an HTTP API, for the developers of the API to realize they did something less than ideal in their resource representations.  As discussed in the [post about representations](/httpapi-resources-representations), this is where versioning comes in.  Defining different versions, however, is only half of the story.  Clients and servers need to be able to agree on the versions they should use when communicating with each other.  That's where [HTTP Content Negotiation](https://www.rfc-editor.org/rfc/rfc9110#name-content-negotiation) (conneg) comes in.

## Response Content Negotiation
If you've heard of content negotiation, it is likely it was focused on how the client and server agree on what response media type to use.  This is called response content negotiation and it's a fairly straightforward process.
1. The user agent includes the `Accept` header in the request.  The value of the header is a whitespace-separated list of media types that client accepts.
2. The server takes that list and intersects it with the list of media types that it supports.  This provides the list of mutually support media types.
3. The server chooses a mutually supported media type.  If there is no mutually acceptable media type, a 406 status code is returned
4. The server encodes the resource into the selected representation and sends it back to the client.  The chosen media type is identified in the responses `Content-Type` header.

This is a very straightforward process that allows the client and server to add and remove supported media types independetly so long as there is always _some_ overlap.  This is crucial in avoiding "big bang" changes in your ecosystem.  Without it, when you had, say, version 2 of your JSON representation come out you would have to synchornize the upgrade of all clients and the server (each of which is probably, in reality, clusters of hosts).  Not fun.

A common question that comes up in step 3 is: how does the server choose from the list of mutually supported media types?  The short answer is "however it wants".  The HTTP spec doesn't require any specific rules.  It does, however, introduce a media type property called `q` (short for quality), that is a floating point number between 0 and 1 (inclusive, however 0 means "not acceptable at all" rather than "least preferred" as you might expect).  This allows the client to provide a preferred weighting to the media types it puts into the `Accept` header.  So, the request might look like this:
```
Accept:  application/json; resource=person, version=1, q=0.5
         application/json; resource=person, version=2
         text/xml; resource=person, version=2, q=0.5
         text/xml; resource=person, version=1, q=0.25
```

This says the client most prefers the v2 of the JSON representation and, if not supported by the server, the client equally prefers v1 of the JSON representation and v2 of the XML representation, finally version 1 of the XML representation is preferred.  Note, these are _hints_ to the server.  The server is **not** required to respect them.

Here is the hueristic I tend to implement in my services:
1. If the request came in via HTTP/1.1 or lower, prefer a text-based representation, otherwise prefer a binary representation.
2. If some media typs have `q` values, prefer the one with the higher value.
3. Prefer the media type with the higher version
4. Prefer the first one in the client's list

From an implementation standpoint, I normally do this by implementing a comparator that is used by a list collection.  I create singleton instances of each for each HTTP version being supported.  I create a readonly list of the server's supported media types at startup time.  Then on each request I take all the media types sent by the client and add them into the list, intersect the list with the list supported by the server, and then take the first element from the resultant list.  There are ways to avoid even the sorted list allocation but the code gets ugly and I do not find the slightly increased effeciency/decressed object allocation worth the difficulty in maintaining the code.


## Request Content Negotiation
Response conneg is great and, when conneg is implemented at all, tends to be where implementations stop.  However, for many HTTP API clients that only covers half of the necessary uses.  Most HTTP API clients want to send representations in HTTP POST, PUT requests (in order to create or update resources).  How does the client know which representations the server will support?  Well, HTTP supports request conneg too.  It works like this:
1. The client sends an OPTIONS request to the endpoint (URL) to which it wants to send the HTTP body content.
2. The server replies with an `Accept` header that lists the media types it supports.
3. The client selects a mutually supported media type.

When selecting the media type, the client can support essentially the same huerisitics as noted for the response negotiation.  The only thing the client usually can not easily determine is all the set of supported HTTP version.  As such, the client will generally just want to assume whatever version was used in response to the OPTIONS request.  This is because the HTTP version negotation tends to be "hidden" from the by the HTTP client library and usually happens at a lower layer in the network stack (generally as part of the TLS negotiation).

Above I said that most conneg implementations don't support request conneg.  So, what do they do?  Well, content negotiation is still going on, it is just happening "out of band".  A given version of a client library simply hardcodes which request representations it will use; maybe with some logic to send text-based representations for HTTP/1.1 and binary representations for HTTP/2 and HTTP/3.  Then, at some point, the HTTP API service announces a new release with a new representation.  The client library is updated to support that version.  Systems using the client library update to the new version and thus switch over to the new representation.  This tends to work out okay most of the time because a given client is usually only working with a single logical instance of the HTTP API service.  Eventually the service will drop support for the older representations so that they don't have to support everything back to the beginning of time.


## Other Negotiations
Generally the developer doesn't have to interact with the other request/response negotiations that happen between the client and server but I am including them here, briefly, for completeness.

`Accept-Charset` lets the sender indicate which character sets it supports.  There is no analogous header indicating what character set is used. Instead it is identified by the `charset` parameter on the `Content-Type` media type.  [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110#name-accept-charset) notes that this header is deprecated because everyone should be using UTF-8 at this point.

`Accept-Encoding` lets the sender indicate which compression algorithm it supports.  The `Content-Encoding` header indicates which encoding was used for the content in the body.

`Accept-Language` lets the sender indicate which human languages, using [RFC4647](https://www.rfc-editor.org/rfc/rfc4647) language identifiers, it supports.  `Content-Language` indicates which language is used for a given HTTP body.  Sometimes, when an HTTP API resource has multi-lingual content within it, developers will want to use this to trim that content down to a single language to send back to the client.  I generally don't prefer this approach because it will force the client to re-issue requests if the user chooses to change the language they are requesting.