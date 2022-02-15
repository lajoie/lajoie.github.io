---
layout: post
title: "HTTP APIs :: HTTP Basics"
category: httpapi
---

One thing that has made HTTP the go-to choice for application protocols is that its basic semantics are very straightforward.  However, this simplicity has also led a lot of developers make explicit or implicit assumptions about how those basic building blocks work.  In this post, I want to look at some of those potential assumptions.

## URLs
The syntax for URLs was originally defined (somewhat poorly) in [RFC 1738](https://datatracker.ietf.org/doc/html/rfc1738) and later both expanded and made more formal in the URI specification, [RFC 3986](https://datatracker.ietf.org/doc/html/rfc3986).  I have some recommendations for the path, query, and fragment components of the URL but there is one thing to get out of the way right from the start.  You should **NEVER** use the [user information](https://datatracker.ietf.org/doc/html/rfc3986#section-3.2.1) portion of the URL.  It is a massive security risk.

The other thing to note is that in HTTP, if a URL resolves, the thing that comes back is a resource (or rather the representation of one).  Always.  This might seem trivial but as I'll cover in the next post, when I talk about "collections" in ReST APIs, this can trip up developers.

### Path
A common problem with the path, query, and fragment portions of the URL is the lack of evaluation rules related to case-sensitivity.  When it comes to paths, I recommend the following for each path segment
* treat them as case-insensitive
* only use alphanumeric characters
* if you must use non-alphanumeric characters, *never* use the forward-slash ('/') character - it is legal to do so but it can cause all sorts of painful-to-debug situations if the character is not properly encoded

### Query
I think the biggest surprise when people first look at the formal specification for "query parameters" is that, in fact, they aren't parameters.  The collection-of-key/value-pairs syntax that we so often see is not how the query component of the URL is defined.  Rather, the specification simply defines it as a single string of printable characters; it has no special structure.  Despite this, I recommend that developers follow the common practice of exclusively using the query component as an ampersand-delimited key/value pair syntax.  This is consistent with what nearly all HTTP libraries expect.  In such a structure I recommend:
* treat each key as case-insensitive
* treat each value as case-sensitive
* always require a value for any given key
* only use query parameters with the GET method (more, below, about using them with DELETE)

### Fragment
Whereas the query string is vaguely defined as "searching" within the resource identified by the path, the fragment is defined as a pointer to a single, specific thing within that resource.  My general recommendation is not to use the fragment within your HTTP API.  If you want to allow a developer to access a subset of a resource then treat that subset as a "sub-resource" and allow the client to address it via the URL.  For example, instead of doing /persons/1234#groups do /persons/1234/groups.  This is also usually much easier for the server-side developers to implement with standard HTTP frameworks.

## HTTP Method Properties
If you've given more than a passing glance at the HTTP protocol you'll know that there are HTTP methods, (often called verbs).  Indeed, there are 8 such methods defined in [section 4](https://datatracker.ietf.org/doc/html/rfc7231#section-4) of the HTTP semantics specification ([RFC 7231](https://datatracker.ietf.org/doc/html/rfc7231)).  They are: GET, HEAD, POST, PUT, DELETE, CONNECT, OPTIONS, and TRACE.  The HTTP specification defines three different types of properties/behaviors such methods might have.  These properties are almost always overlooked or, when considered, often conflated.  However, they control how various network intermediaries (proxies, CDNs, etc.) expect requests to operate and having a disconnect between your service implementation and these behaviors can lead to a lot of headaches.

"Safe" methods (GET, HEAD, OPTIONS, TRACE) are semantically read-only methods.  That is, no matter how many times a client calls them or with which parameters, headers, or body, the state of the resource being addressed does not change.  This means clients (either end-user or network intermediary) may safely retry requests without user intervention.  On the server side, the response may be cached until the server specifically does something, independent of the client, to change the resource state.

"Idempotent" methods (PUT, DELETE)<sup>[\*](#foot1)</sup> are "desired state" methods.  That is, the methods express the client's wish to alter the current state of a resource to a given desired state.  Because a request provides the desired state, in its entirety, these methods may also be blindly retried; the first invocation might change the resource state but subsequent ones do not.  If the state of the resource is as requested, whether because the server just changed it or because the resource was already in that state, the request is considered successful.

"Cacheable" methods (GET, HEAD, and POST) are, as the name suggests, safe for clients and intermediaries to cache.  The HTTP specification also notes that while POST is technically cacheable, almost no HTTP implementation treats it that way.  I'll discuss caching more in another post but as a spoiler, I think caching in the HTTP layer, within the context of APIs, is best avoided.

## HTTP Methods

### DELETE
The DELETE method is defined to do pretty much what it says on the tin - delete the resource identified by the URL.  However, there seems to be two main things that developers get wrong when implementing this method.  The first is the status code.  The HTTP specification says:
> If a DELETE method is successfully applied, the origin server SHOULD send a 202 (Accepted) status code if the action will likely succeed but has not yet been enacted, a 204 (No Content) status code if the action has been enacted and no further information is to be supplied, or a 200 (OK) status code if the action has been enacted and the response message includes a representation describing the status.
> \- [Section 4.3.5](https://datatracker.ietf.org/doc/html/rfc7231#section-4.3.5), RFC 7231

Note that 404 is not mentioned.  This is because DELETE is an idempotent method, i.e., it's expressing a desired state.  So, if a client makes a DELETE request for a resource the server no longer has, then the server is already in the desired and so the request is successful without any further processing.

The second issue arises when using the DELETE method on a collection resource.  For example, maybe a developer wants to provide the ability to delete people that are inactive and so wants to support a call like this:  DELETE /persons?status=inactive.  This seems reasonable enough.  The big problem arises from how HTTP frameworks generally handle query parameters: unknown parameters are ignored.  So, in my example, if the client used a parameter name of 'state' rather than 'status' this would be equivalent to 'DELETE /persons', i.e., delete the entire person collection.  So, generally speaking, I recommend against supporting DELETE on collection resources.  If you do, the server should return an error if:
 * The request contains a query parameter the server does not understand.
 * The request would result in deleting more resources than a server-defined value.  The server may also allow the client to provide a *smaller* number via a query parameter.

 If the server does support DELETE on resource collections, the server and client must be able to handle the possibility that the request is only partially successful.  In my example above, perhaps there are five people in the inactive state.  But, due to other business rules, only three of them can be deleted.  What does the server respond to the client with in this case?  There is no one right answer but it's a case for which both sides have to be prepared.

### PATCH
The PATCH method is an extension to the HTTP protocol defined in [RFC 5789](https://www.rfc-editor.org/rfc/rfc5789.html).  The idea is that instead of sending the entire representation, as required by the POST and PUT methods, a client can send only the expected differences instead.  The reason I hear most often cited for doing this is a wish for smaller payloads.  Unfortunately, this method is one of those things that sound good on paper but are pretty disappointing in reality.

The first problem is that RFC 5789 doesn't actually specify a representation for a patch.  [RFC 6902](https://datatracker.ietf.org/doc/html/rfc6902) specifies such a representation for JSON but that is only one of literally enumerable representations that could be supported by a server (and I am a big believer in supporting multiple representation).  So, for everything else you're forced to come up with your own patch format which in turn means every client out there is going to have to implement their own support for your proprietary format.  This likely means a never-ending procession of edge cases you didn't consider.

The second problem is with the actual implementation of patches on the server side.  Patches are evaluated in the context of, and applied to, the *representation* of a resource.  So the server needs to pull the current resource, encode it in to a representation, apply the patch, decode the patched representation and only then proceed with evaluating the updated resources as it would on a more standard PUT.  The extra work there isn't great from a performance/scaling perspective.

Finally, and as a corollary to the previous problem, you will need to consider potential concurrency issues when patching.  These arise when the client generates a patch based on the state of the resource as they currently have it and then send it on to the server.  When the server does its dance to apply the patch, it might be applying it to a resource that other clients have already updated.  This can be detected and addressed within POST and PUT methods, as I'll discuss in a future post, but the PATCH method doesn't have such a mechanism.

At the end of the day, if the reason you want to use PATCH is to decrease request sizes, my recommendation would be to move to HTTP/2 (or HTTP/3 when available) and use a binary representation (e.g., protobuf, avro).  In most cases you'll end up with smaller payloads than with PATCH and have much cleaner code on the server side.

### POST
The POST method has a very open-ended definition:
> The POST method requests that the target resource process the representation enclosed in the request according to the resource's own specific semantics
> \- [Section  4.3.3](https://datatracker.ietf.org/doc/html/rfc7231#section-4.3.3), RFC 7231

In short, it's up the resource, identified by the URL, to determine what it means to "process" the request.  In ReST APIs, the only resources that generally have a notion of "processing" a request are collections (e.g., the resources you'll find at pluralized URLs like /stores, /persons) and that "processing" is almost always "add to the collection".  For RPC APIs, which should be using POST as their primary HTTP method, the notion of "processing" is "invoke the method identified by the URL".

A behavior to consider, in ReST APIs, when creating a new resource via the POST method is whether the server or client should assign the ID of the resource.  The general practice is to have the server do it and indeed some frameworks have this behavior hard coded into them.  However, the HTTP spec does not in fact require this behavior (though it does give an example of it as a possible behavior).  Either way, it's important that the server provide a descriptive error message if the expected behavior is violated; if the ID already exists when the client is assigning IDs or if the client provides an ID when the server is assigning IDs.

Another ReST-specific behavior to consider is whether a client should be able to create multiple resources in a single POST (i.e., provide a collection of resources in the request).  Many frameworks do not support this and in general my recommendation is that developers avoid it.  Like with the DELETE method you run into the problems of how to report partially successful requests.  It's not that doing so is impossible but it is complicated both on the server- and client-side and thus more prone to mistaken implementations.

### PUT
The PUT method has a much more prescribed definition than POST:
> The PUT method requests that the state of the target resource be created or replaced with the state defined by the representation enclosed in the request message payload.
> \- [Section 4.3.4](https://datatracker.ietf.org/doc/html/rfc7231#section-4.3.4), RFC 7231

So, like with DELETE, PUT is a "desired state" (i.e., idempotent) method.  It simply, and exclusively, replaces the resource at the given URL with the resource in the body of the request.  That's it.  Nothing else.

A thing often missed by developers is that PUT can be used to *create* resources.  So service developers should consider whether they want this behavior or not.  If not, this needs to be explicitly called out.  If the resource-creation behavior is desired, please note that it is the clients that will be assigning IDs to the resource.

The other behavior to consider is whether to allow PUT calls on collection resources (e.g., PUT /persons).  Such calls would mean "replace the entire collection with the one found in the body of this request".  I think in most cases this is not the behavior most clients would want so, in general, I recommend that service developers not support this.  It can, however, be useful in certain niche cases and so is helpful to be aware of as a possibility.

## HTTP Headers
When it comes to headers my main rule is: "no custom headers" or at least none that are required for processing a request; those used to carry optional information (e.g., request tracing data) are fine.  The problem with custom headers is that some network intermediaries will mangle or drop them.  Given that most intermediaries are outside the service provider's control there isn't much the provider can do to address this issue.  Worse, it may be that today every intermediary deals with your custom header just fine but, due to a new intermediary coming online or an upgrade to an existing one, tomorrow there is a problem.  In such a circumstance, the service provider and every client is going to have to scramble to update their code.  In my opinion it's best to avoid this potential problem right from the start.

The other header-related item I'll note here is that for decades the X-Forwarded-* headers were used to carry things like the requesting client address, the hostname the client used in their request, and some other things when those pieces of data are changed by intermediaries like load balancers and proxies.  These headers, as signified by the X- prefix, were experimental but they became a *de facto* standard.  During the creation of the HTTP/1.1 RFC 7320 series this functionality was formally standardized in [RFC 7239](https://datatracker.ietf.org/doc/html/rfc7239).  Make sure your application servers are giving preference to the now-standard header.  It has cleaner semantics than the experimental ones.

## Status Codes
Status codes are another one of those HTTP items that everyone "knows all about".  I think here I just want to call out two things.  First, there are a number of 4xx and 5xx codes that can indicate more specific errors than the common 404 and 500 errors.  Review them and use them in order to provide a more robust error indicator to clients.  Second, stop using the 301 and 302 redirects.  There are various security problems with them and they have been superseded by the 308 status code defined in [RFC 7538](https://datatracker.ietf.org/doc/html/rfc7538)

---
<sup><a name="foot1">\*</a></sup> The HTTP spec does also note that safe methods are idempotent methods.  I think doing this makes the most salient difference, the "desired state" nature of the idempotent methods, harder to grasp.  The technical reason for including the safe methods is that, because safe methods don't change the state of the resource at all, the resource is technically in the desired state at the end of the request.  This has always felt like confusion-inducing, hair splitting to me.