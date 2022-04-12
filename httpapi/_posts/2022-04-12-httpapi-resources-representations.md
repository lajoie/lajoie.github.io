---
layout: post
title: "HTTP APIs :: Resources And Representations"
category: httpapi
---

At the core of HTTP is the notion of resources and representations.  In this post I'll take a deep dive into each as well as discuss some implementation details that are important to consider when implementing an HTTP API.  My next post will then discuss how some of the functionality and recommendations described here are used within the HTTP requests and responses in order to get some desirable behavior.

## Resource
Within the HTTP specifications, the concept of a resource has a very simple definition: the thing to which a URL points.  A resource might be an image, an audio track, a record of an individual person, etc.  However, within the space of HTTP APIs, a resource almost always boils down to a named collection of key/value pairs (e.g., the aforementioned person record).  Building off of that are what I'll call collection resources which are resources that are primarily a collection of other resources (e.g., the collection of all orders in the system).  These are often, but not exclusively returned when performing a search.

The key/value pairs in the resource are often referred to as fields or properties of the resource.  Like fields in an OOP class, the resource field may be required or optional.  The values of the fields may be constrained in various ways (e.g., required to be of a certain type, within a certain range, unable to be changed after being defined, etc.).  The value of a field may also be another resource, including a collection resource.  That is, resources may be composed.

### Resource Identifiers
As noted above, what makes something a resource is that a URL, specifically its path, points to it.  Such a URL, then, must contain some piece(s) of information that reference the resource, i.e., an identifier.  This identifier is almost always just a single piece of information but it is possible to have a composite identifier.  Let's start with the identifier for a collection resource, because it's the easier of the two, and then go to the identifier for the non-collection resource.

The identifier for a collection resource is almost always just the name of the resource type in the collection in some pluralized form (e.g., carts, products, tools).  If the noun would normally have an irregular pluralization, I recommend using the regular plural form (e.g., persons not people) as it will likely match the name of an object within your codebase.

I recommend using a synthetic, opaque, randomly distributed identifier for non-collection resources.  Let me break down each of those adjectives.  'Synthetic' meaning that the identifier field has no meaning other than as an ID for the resource.  No matter how certain you are that any other given field will never change, if your service is around long enough, there is a very high probability that some use case will surface that requires you to change that field and changing an ID is painful.  'Opaque' meaning that there is no encoded meaning in the ID, it's just a random string.  IDs show up in URLs and URLs show on screens and in logs and these are places you generally don't want potentially sensitive information showing up.  'Randomly distributed' means that you are (roughly) equally likely to get any value over the entire value space for the identifier.  This prevents a malicious actor from being able to look at the ID and surmise aspects about how IDs are assigned and potentially focus in on certain ones to attack.  I often use a [Type 4 UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_(random)) for resource IDs because they are easy to generate (every language I've encountered has a method/function that "just does it"), and they are random enough that the chances that you are going to generate identical IDs is effectively zero (though your code still has to account for the possibility), and fast enough to generate that they don't pose a performance problem unless you're generating a lot (tens of thousands) in a short time (subsecond).

## Representations
As already stated a few times in this post and previous ones, representations are a way of expressing a resource within the context of requests, responses, files, etc.  The process by which a resource is converted into a representation is generally called encoding while the reverse process, taking a representation and turning it into a resource, is generally called decoding.  A given resource can have multiple representations and, if the service has been around long enough for business needs to evolve, often will.  A given representation may express every piece of information known about a resource or only a subset.  The way a piece of information is expressed may also change over time as business needs evolve.  All this is to say that a representation may evolve separately from the resource it is expressing and a corollary to this is that a resource must contain a superset of all the data expressed in all its representations.

When encoding a resource, there are two fairly important aspects to the representation - its structural type and its schema.  The structural type of a representation is the general format for the representation (e.g., JSON, XML, Avro, protobuf, and [ASN.1](https://letsencrypt.org/docs/a-warm-welcome-to-asn1-and-der/) \[I put that in there just for some of my friends for whom those five characters is the stuff of nightmares; you're welcome]).  A developer may think of the structural type as "the parser I use to read the representation".  The schema is the definition of how specific resource fields are represented in the structural type (e.g., the first name field is a property of the root JSON object with the name 'first_name' and the value is a non-null string).  Some structural types (e.g., XML and protobuf) define a formal language that can be used to express the schema while others require developers to simply write an informal, expository description as I did in my previous example.

### Media Types
A representation is identified by a [MIME type](https://datatracker.ietf.org/doc/html/rfc2045), now more often referred to by the more generic name "media type".  A media type contains four parts: a primary type (required), a subtype (required), a suffix (optional), and a collection of parameters (optional).  The primary type is a [set of values, ten currently, controlled by IANA](https://www.iana.org/assignments/media-types/media-types.xhtml) while subtypes are usually defined by a standards body and then registered with IANA.  You've almost certainly seen many examples: `application/json`, `text/html`, `image/jpg`, `audio/mp4`, etc.

When it comes to identifying the representation of a given resource, as noted above, you want to be able to identify both the structural type and the schema.  Or saying it another way, you want to be able to answer the questions "what parser do I use?" and "what data should I expect to find after parsing?".  The general primary/subtype media types only answer the first question (e.g., if you see `application/json` you know to use a JSON parser).  The second question must be answered from other, indirect contextual information; a process that is often prone to error.  So, instead I recommend using the other parts of the media type to directly answer that second question.  There are a few different ways to do this but over the years I have found this to be best: define a parameter that explicitly identifies the resource.  

This parameter alone can give rise to days/months/years of [bikeshedding](https://en.wiktionary.org/wiki/bikeshedding).  Here is where I've settled.  Use `resource` for the key - it's clear and simple.  Use a period-separated, namespace-qualified resource name for the value.  For the namespace I start with something that identifies the company I'm working at (e.g., `dominos`, `hagerty`), then follow that with a bounded context name (in the Domain Driven Design sense) and then finally the resource name.  Note, that in DDD systems, it's not uncommon for there to be an entity whose name matches the name of the bounded context. This can lead to cases where the last two parts of the `resource` value are the same (e.g., `dominos.order.order` or `hagerty.person.person`).  This can look odd at first but is pretty easy for people to "get" when you explain it to them.  I strongly recommend _against_ including any sort of department or product name in the namespace; those change too frequently.  Taken together, this leads to a media type that looks like `application/json; resource=hagerty.person.address`.

### Versioning
If you somehow managed to avoid shutting your entire project down for a year to debate the name and value of the `resource` parameter, let me give you another chance with...  versioning! Once you've defined a representation you need to consider whether it might need to change over time and whether, in the event of such changes, you will require all clients to upgrade at the same time as the server.  If your HTTP API is exclusively meant to be consumed by a UI that you also own it's probably not a big deal to synchronize the updates.  If, however, there are multiple users of your HTTP API and those users are outside your group, or maybe even outside your company, the blast radius of a change to a representation is much greater and the tolerance for a "big bang" upgrade is going to be much lower.  This is where versioning comes in.

In order to deal with the fact that a given representation is likely to need to change over time, I recommend that you version it.  There are various ways to represent this version in the media type but I recommend it be carried by another media type parameter.  I call the parameter `version` and its value is a single, incrementing integer that behaves as the major version in the [semantic version model](https://semver.org/spec/v2.0.0.html); that is you increment the version number when you make a change that is not backwards compatible.  Such non-compatible changes are:
* The introduction of a new, required field.
* Changing an existing field to a wholly different type (e.g., string to array), a more specific type (e.g., numeric to a float), or adding or changing restriction rules (e.g., the value can only be 50 characters long now).
* The removal of a field.

Once I reference the major version number of the semantic versioning spec I often get the suggestion that the minor version should be used as well so that clients can glean extra info about additional, optional fields that the representation may support (and it can only reference optional fields because if changes were made the required fields it would be a breaking change).  I recommend against this for two main reasons.  First, developers start treating the fields as non-optional; "I see this is version 1.3 so it will have field X".  Second, developers forget that other clients may not have upgraded yet and so don't know about or understand the field.  In my next post I'll also get into another reason why this tends not to work very well.

### The vnd. Media Type Form
As mentioned above, there are various ways one could represent the resource type and version of the representation.  An alternative way to do this in the media type is to use the [Vendor Tree](https://datatracker.ietf.org/doc/html/rfc6838#section-3.2) subtype and suffix.  In this model, the resource type identifier and version get encoded into the subtype while the structural type is placed into the suffix (e.g., `application/vnd.hagerty.person.address.v1+json`).  I used this form for a long time but it requires fiddly escaping and parsing of the subtype that eventually I just wanted to avoid.  It's much easier to access parameters via some associative array (i.e., dictionary, map) API.

### Representations In The Wild
So far, all this talk about resources and representations has been framed in the context of HTTP APIs but, that is not the only place where this is useful.  Once the infrastructure is in place to do this work, I have found it quite useful elsewhere.  One such place is in a messaging layer and it's for the same reason as in the HTTP API - allowing client and server to know how to parse and what to expect after the parsing.

A more interesting use case though came in the data persistence layer of a project I worked on.  In that project we had to use SQL Server as the backend.  We naturally started with an ORM approach because that is just where you start nowadays (as an aside, I hate ORMs in every language I've used them in).  However, our object models were pretty complex and were changing frequently because our services were new.  It became really time consuming to add new fields in that model (adjusting table schemas, migrating data, dealing with failure cases to both).  Plus response times weren't great with all the joins going on (most of our use cases needed the fully hydrated resource).  After a while we took a step back and asked ourselves why we were forcing our resources into this relational model.  The space savings that might be afforded from normalizing the schema was pretty low.  We also didn't expose any support for _ad hoc_ queries.  So what we ended doing was, for each resource create a single table and:
* pick out the couple things we did need to query on, place them into separate columns, and index those columns
* encode the resource into a representation and stuff it into a CLOB column and put the media type into another column

This pseudo-document DB method proved very beneficial for us.  Supporting new fields became just a matter of adding it to encoding and decoding logic which we normally had to do to support the HTTP requests/responses anyways.  We were also able to do "rolling updates" of our data - as records were read we decoded with an older decoder (which we looked up via the media type) and as records were written we encoded with the latest encoder.  We also saw a big performance increase by getting rid of joins.  We decided to use the JSON structural type for our content because SQL Server supports querying into JSON documents, albeit much more slowly than querying into a relational model, which allowed us to do _ad hoc_ queries in the rare cases where we need them; usually for investigative purposes.  The one downside was that once it became trivial to put stuff into the resource developers wanted to start putting all sorts of random stuff in there; we had to review resource changes carefully.

## Resource Implementation Considerations
Here are a collection of recommendations I have around the code-level implementation of resources.

### Resource as a Representation Superset
As noted before, and as should be intuitively obvious, the resource must contain all the data that will be presented in any representation of that resource.  Where else would it come from if not the resource.  However, this does not mean that the data in the resource must be structured in exactly the same way as it is in the representation.  For example, if the resource object has an address field whose type is an Address object with its own set of fields the representation could still have an address field whose value was a single string.  It just means that during the encoding/decoding of the object there needs to be a lossless way to go from the Address object to a string and back.

I often see the anti-pattern where, for each logical piece of data, a developer will add a field to the resource for each unique way it is represented.  Don't do that.  Encapsulate encoding/decoding logic in encoder/decoder classes don't smear it over your resource objects.

### The DTO Anti-pattern
Another anti-pattern that I often see, and it's related to the previous one, is the creation of DTO (data transfer objects) for each supported representation.  From an efficiency standpoint, these are pretty bad.  During decoding you will likely end up parsing the representation into a tree of generic objects supported by the parser, copying that data to DTOs that you create, and then copying that data into the underlying resource object.  Count that up and you have four copies (the pre-parsed representation, the parser objects, the DTO objects, and the resource) in memory.  Reverse that for encoding.  Depending on how big your representations are and how many concurrent requests you expect this could chew up a lot of system resources.

You can eliminate one copy of the data by copying directly from the parser-generated objects into the resource.  This will often mean writing your own encoder/decoder classes (so, no DTO classes).  You could eliminate another copy if your parser supports an eventing or "push" model (so, no tree of objects from parsing).  Finally, you can avoid having the entire "raw" representation in memory if your parser supports streaming.  I personally start at the second step (not using DTOs), because it's fairly easy to do and I only move on to later options if I really need to optimize something and I'm willing to pay for it with the increased complexity that eventing and streaming introduce.

### Avoid Auto-Binding Libraries
When talking about encoding/decoding resources/representations many developers will immediately reach for binding libraries.  Binding is the fancy name for "library that automagically does encoding/decoding for me".  There are a various drawbacks with these libraries that make me recommend against them:
1. They are complicated.  Often they are doing a bunch of reflection/metaprogramming to get data in to and out of the resource objects.  If something doesn't map appropriately, it can be difficult to tell why and fix it.
2. They tend to be really slow.  Because they are using reflection they are often orders of magnitudes slower than hand-written code.  In order to avoid this slowness, in languages that support it, the binding library will often self-generate and load additional code that tries to mimic what a developer might have written.  This though further increases the complexity.
3. It can be hard to impossible to transform values as they are written out.  For example, if one of the field values should contain a link to another resource you will want to inject the deployment-specific hostname.  Most binding libraries don't provide a way to do that.
4. No binding framework, that I know of, supports different versions of a representation.  For example, v1 wants the field represented as a coma-separated string while v2 wants the data represented as an array.  This tends to lead people back to DTOs.

### Avoid Automatic Type Coercion
Many parsing libraries and binding frameworks support type coercion (e.g., taking a string and asking it to be converted into a boolean or taking a string and asking it be converted into a datetime).  I recommend avoid any automatic type coercion.  For one, most devs aren't clear on the full set of coercion rules and those rules differ from library to library.  For example, some libraries will convert any string that isn't "true" or "T" into false.  Is that what you actually want/expect?  Second, when type coercion fails the errors returned by the library are often not overly helpful/informative.

## Representation Implementation Considerations
There are various common patterns/needs/mistakes I've seen show up again and again within representations.  Here are the big ones.

### Field Standards
Define constraints/rules for how basic data types will be encoded in your representations:
* Use native types when possible.
  * Use booleans rather than strings or numbers
  * Use numerics rather strings 
* For strings
  * Trim any leading/trailing whitespace
  * Omit encoding empty strings unless they have a particular semantic meaning.
* For dates, times, and datetimes use string values in the ISO8601/[RFC3339](https://datatracker.ietf.org/doc/html/rfc3339) format (i.e., YYYY-MM-DD, hh:mm:ss.sssZ, YYYY-MM-DDThh:mm:ss.sssZ, respectively) normalized to UTC.
* Omit any null-valued fields from the representation unless the structural format requires them and then use whatever its native null value is.
* Prefer associative arrays (i.e., dictionaries, maps) over numerically indexed arrays as most libraries make it easier to address data by the keys of the former than the a numerical indexes of the latter.

Determine and document which fields may be set only when the resource is created (init-only fields), at any time (read/write fields), or never (read-only fields).  If requests pass in init-only or read-only fields on a change request, the service should ignore them.  They are likely being passed in only because the client received them back from a query and took that response as a starting point to which it has applied changes.

### Metadata
In addition to whatever normal fields make up a resource, it is good a idea to define a field that can carry system-related metadata.  I use the field `meta` for this.  Exactly what is contained within it is naturally system-dependent but some common data are who and when the resource was created and updated.  Other fields will be discussed elsewhere in this series.

### Collections
It's not uncommon for certain endpoints to return collections of resources.  Such collections are, themselves, resources and representations of those resources should have their own media type.  For the value of the `resource` parameter of the media type I get really inventive with the naming and add `Collection` to the end of the name of the contained resource, e.g., `application/json; resource=hagerty.person.addressCollection`.  I use `Collection` because it does not imply ordering, type, uniqueness, or nullability of elements all of which vary based on the structural type.  [JSON arrays](https://datatracker.ietf.org/doc/html/rfc8259#section-5) for example are unordered, untyped, non-unique, and support null elements while [protobuf `repeated` fields](https://developers.google.com/protocol-buffers/docs/proto3#specifying_field_rules) are ordered, typed, non-unique, and do not support null elements.

When it comes to collections you will often want your HTTP API to restrict the number of entries it will return in a single request; e.g., you don't want some client trying to pull back 5 million person records in a single request.  That is to say, you want to paginate the returned collection.  To represent this, I recommend the following fields within the collection's `meta` field:
* page - The page number of the collection being represented.
* pageSize - The size of pages being returned by the service.  This isn't strictly necessary but it helps makes explicit the page size that a service will use if a client doesn't request a specific page size or if the client requests a page size larger than is allowed by the service.
* hasMore - A boolean value indicating whether there are more pages.  This isn't necessary but it helps UIs know whether they should display an indicator that there are more pages.  A common, efficient way to implement this is to simply request of the backend storage system a result set size of one more than the page size.  If that final record comes back you can strip it off and set hasMore to true.

A common request is to also carry the total number of possible pages.  I do not recommend doing this for two reasons. First, it can be expensive to compute.  Second, it can change over time while viewing the paginated results and many UIs don't gracefully handle the total number of pages changing over time.

### Content Translation & Localization
There is a good chance that some resource in your system will contain text for which, ideally, there are multiple translations available.  The solution you would use for static content on a page in your app isn't going to work here because the resources in your system will be dynamic (e.g., someone adds a new product to your app and you need its description in three different languages).  To address this, I use a pretty simple map-of-maps structures.  The key to the first map is the ISO 15897 _language___territory_ locale identifier (e.g., en_US, de_CH) and the value is a map whose key is an identifier for the translated content (e.g., description, smallLogo) and the value is the translated content.  Note, the translated content need not be text.  For example, it could be a URL to an image that contains text that is translated (e.g., if you have a product image that contains the translated product name on it).  I use the term "display items" for the overall structure, "display item language" for the first key, "display item ID" for the second key, and "display item value" for the value associated with the second key.  Here's an example:
```
DisplayItems: {
	"en_US": {
		"name": "Quarter Pounder® with Cheese",
		"description": "Each Quarter Pounder® with Cheese burger features a ¼ lb.* of 100% fresh beef ..."
	},
	"de_CH": {
		"name": "Cheeseburger Royal",
		"description": "Der Königliche, der dich dahinschmelzen lässt."
	}
}
```

I use this structure for a couple different reasons.  First it collects all of the translatable content for a given resource into a single field, which I usually call `DisplayItems` rather than having a new field for each display item ID.  This means that if you provide a way for a client to include/exclude specific fields in a response, the client can include/exclude all translatable content at once depending on its needs.  I key things off the language first because chances are you'll want everything in a single language rather than something like the product name in English and the product description in German.

When it comes to the localization of numbers, I don't do it.  At least not on the server side.  I think the server should provide data in a single standard format and the client should reformat the data, for the required locale, as it is rendering it.  Clients known which locale they want to operate in and usually have richer libraries for this type of formatting than servers do.

### Error Representations
Deciding what to return when there is an error often isn't given much thought by HTTP API designers.  This is unfortunate as it is the primary mechanism by which clients may troubleshoot a problem.  For me, I ended up with a relatively simple error resource (yep, errors are resources too).  The error resource contains:
* correlationId - The [tracing ID](https://www.w3.org/TR/trace-context/) associated with the request.
* location - The endpoint to which the request was sent which is helpful for logging.
* status - The HTTP status code.  I don't like replicating data that could be found in the HTTP response itself but I still encounter libraries that make this hard to get at in error cases.

Then the error resource has an `errors` field that is an array of error objects that contain:
* code - A unique identifier for the type (not this particular instance) of the error.
* description - A human readable description of the error suitable for logging but NOT for displaying to the end user.
* data - An associative array (a.k.a. dictionary, map) of contextual data.  All keys and values should be strings.

When a client wants to display an error message to the user it can, for each error returned, use the error's code to look up a displayable error string from a translation bundle.  I recommend that these error strings be templates which are populated with data found in the `data` field.  So instead of displaying "An item in your cart is no longer available" you could display "The item 'KOUGU Super Precision Long Needle Nose Pliers' is no longer available".  It is my opinion that the client is best positioned to know the languages it needs to support and the wording it wants to use in the context of the rest of the application and so it should control the actual message displayed not the service.

Here's an OpenAPI Specification for this error representation:
```
errorResponse:
  type: object
  description: An error response to a request.
  properties:
    correlationId:
      description: The tracing ID for the request.
      type: string
      example: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
    errors:
      description: >
        The errors that resulted from the request.
        Note, there may be multiple errors with the same id within this array if the same error occurred
        multiple times.  For example, if the error represents a generic input validation error and three
        fields failed to validate than you would see three error elements with the same ID.
      type: array
      items: #/errorElement
      minItems: 1
      example: [ 
        {
          "id": "nkjg3jbgew",
          "data": {
            "cartId": "2b7389a4-ee29-4d78-aadc-730e45c6f549",
            "item.id": "d1e956fd-646c-4b65-ac84-4424c603e8d6",
            "item.name": "KOUGU Super Precision Long Needle Nose Pliers"
          },
          "description": "item in cart is no longer available"
        }
      ]
    location:
      type: string
      description: The path to which the request was sent.
      example: /cart/f3b5f249-ae3f-40db-9bdd-9604c871bf25
    status:
      type: number
      description: The HTTP status code of the response.
  required:
    - correlationId
    - errors
    - location
    - status

errorElement:
  type: object
  description: An individual error that resulted in the failed request
  properties:
    id:
      type: string
      description: An identifier for this type (not instance) of error.
  data:
    type: object
    description: >
      Key-value pairs providing contextual data associated with the error (e.g., the ID of a cart item). 
      Both key and value must be strings. 
      Almost all errors should have contextual data. If you are creating one that does should stop and 
      reconsider whether there is some data you should be including.
    additionalProperties: true
  description:
    type: string
    description: >
      A human readable, non-internationalized, description of the error.
      This is appropriate for logging but not for display to an end user.
 required:
   - id
   - description
```

