class: center, middle

# IIIF Authentication at UCLA <br> in 2023

**Mark Matney**

Code4Lib SoCal : July 14, 2023

.ucla-gold[
---]

![QR encoding of slides URL][slides]

???

* **_put cloned window in presenter mode_**
* for those who don't know me:
    * Mark @ UCLA Library SDLS / Services Team
* going to talk about our IIIF Authentication API 1.0 implementation "Hauth"
    * (I think) pronunciation is the same as its cultural reference
    * if you think you've got the ref, you should let Joshua know at happy hour ;)
* roadmap:
    * review concepts
    * outline our motivation (why we need IIIF Auth)
    * implementation
        * requirements
        * development process
        * deployment status
        * evaluation
        * light implementation details
    * a bit about IIIF Auth 2.0

---

## Concepts

* **IIIF:** API specs for _interoperable software to represent/deliver/render digital artifacts on the web_
    * Images, books, audio recordings, movies, 3D models, arbitrary RDF graphs (someday 🫠)

* **IIIF Auth:** interoperable access control
    * Describes behavior of both client (viewer) and servers **to coordinate secure, authorized transfer of description resources (info.json) and content resources (images)**

???

* _poll_:
    1. Who is familiar with IIIF? Who (besides UCLA) is at an institution running IIIF software in production?
    2. Anyone running IIIF Auth in production? Anyone experimented with it?
* IIIF: Mirador, Universal Viewer, Cantaloupe, Loris
* IIIF Auth:
    * we implemented 1.0
    * a solution following browser security rules is non-trivial!
    * version 1.0 extends IIIF Image API (conformant image servers must implement Auth spec)
    * fine granularity
    * it works, but:
        * relies on web browser behavior re: third-party cookies that is being phased out by major vendors
        * is not generic/abstract enough for some use cases

---

## Motivation

Our users:

* **UCLA Library staff**
  * Site administrators
  * Specialized- and collection-focused access

* **Non-Library UCLA staff who collaborate with the Library**

* **UCLA students and faculty**
  * Enrollment status
  * Course membership

* **Campus network users (including guests)**
  * UCLA VPN

* **Bespoke research website users**
  * Sinai Manuscripts Digital Library

???

* _why does UCLA Library need access control for some of its content resources?_
* answer (complicated): copyright, money, etc.
* answer (simpler): we have identified lots of materials for which access is conditioned upon membership in a particular user group

---

## Requirements for v1

Our users:

* .gray[**UCLA Library staff**]
  * .gray[Site administrators (e.g. Library Digital Collections)]
  * .gray[Specialized- and collection-focused access]

* .gray[**Non-Library UCLA staff who collaborate with the Library**]

* .gray[**UCLA students and faculty**]
  * .gray[Enrollment status]
  * .gray[Course membership]

* **Campus network users (including guests)**
  * UCLA VPN

* **Bespoke research website users**
  * Sinai Manuscripts Digital Library

???

* for v1: targeted simplest, most well-defined methods of determining user group membership
* e.g., don't deal with SSO, MFA schemes, etc.

---

## Development timeline

**Start:** Summer '21

**MVP finish:** Fall '22

**In production:** Spring '23

Deliverables:

* https://github.com/UCLALibrary/hauth

* https://github.com/UCLALibrary/cantaloupe-auth-delegate

* https://github.com/UCLALibrary/docker-cantaloupe/tree/main/src/main/docker/patches

* https://github.com/UniversalViewer/universalviewer/issues/860

???

* code repositories
    * in addition to server app, also needed to implement parts of the spec in Cantaloupe
* we finished the MVP almost a year ago, but were held up by implementation gaps in third-party apps
    * our target IIIF viewer (UV) didn't fully implement the 1.0 spec
        * for good reason: plans of browser vendors to tighten restrictions on third-party cookies known when we began
    * Cantaloupe also lacked functionality
    * the "Interoperability" I in IIIF is true, in theory
* by Spring '23 we had begun to publish restricted materials

---

## Deployment status

User group impl | Status | Items affected
--- | --- | ---
Campus network | ✅ In use in production <br> 📫 Publishing content | ~10<sup>1</sup> images (July 2023) <br> eventually ~10<sup>4</sup>
Bespoke research website | 🏚️ Bespoke auth remains | ~355,000 page images from 923 Sinai MSS <br> (Spring 2023)

???

* _how are we using Hauth at UCLA?_
* so far: on the order of tens of images published (previously unpublished) that are restricted to campus network users
    * exactly: 33-page document
    * tens of thousands queued up
    * several 1,000+ item collections, many smaller collections
* also so far: yet to migrate published Sinai materials to IIIF Auth
    * lower priority
* "derelict house building" emoji XD

---

## Evaluation

User group impl | Rating | Comments
--- | --- | ---
Campus network | .yellow[★★★].cyan[★★] | 👍 Configurable for any IPv4 network spec <br> 👎🏻 Same method of degradation applied to all images <br> 👎🏻 Single degraded tier; all-or-nothing access not allowed <br> 👎🏻 Doesn't support IPv6 network spec
Bespoke research website | .yellow[★].cyan[★★★★] | 👎🏻 Must be implemented for each research site <br> 👎🏻 Extremely brittle

???

* campus network users (IP address)
    * configurable for any IPv4 network specification
    * same method: size reduction ratio
    * single tier, all or nothing not possible
    * no IPv6 support
* researchers (bespoke auth)
    * just pretty bad
    * probably could have implemented in a more general way
    * Auth 2.0 seems to be more clear on how to implement this use case

---

## Example content restricted to campus network users

.url[https://digital.library.ucla.edu/catalog/ark:/21198/zz002dwzpk]

!["An Experiment in Data-Processing Shared by Two Computers": UCLA Library Digital Collections][item]

  [item]: ./item.png

<img src="./item-qr-code.svg" style="position: absolute; right: 100px; bottom: 50px">

???

* live example: 33-page document
* try accessing it both on and off a UCLA network (VPN or campus access point)

---

## Example content restricted to bespoke research website users

.url[https://sinaimanuscripts.library.ucla.edu]

![Sinai Manuscripts Digital Library search results: all manuscripts][sinai]

  [sinai]: ./sinai.png

???

* access to these materials is still determined by the site's bespoke authentication mechanism

---

## Implementation

* **Java 17** with **Maven** (builds **Docker** image)
* **Vert.x** (async types) → https://vertx.io

```java
static Future<Set<Row>> getFieldValue(String fieldName, int identifier, Pool dbClient) {

    // Async expression
    Future<SqlConnection> getConnection = dbClient.getConnection();

    Function<SqlConnection, Future<Set<Row>>> executeQuery = connection -> {
        var queryTemplate = "SELECT $1 FROM table WHERE id = $2";

        // Async
        return connection.preparedQuery(queryTemplate)
                         .execute(Tuple.of(fieldName, identifier));
    };

    // Async
*   return getConnection.flatMap(executeQuery);
}
```

The old way:

```java
static void getFieldValue(String fn, int id, Pool dbClient, Handler<AsyncResult<Set<Row>>> h) {
    /** Send caller into callback hell and/or NPE pitfall city */ }
```

???

* Vert.x
    * library / application framework that we've used on several projects at UCLA Library
    * basically: async types for building reactive, event-driven JVM apps
        * performance
    * contains:
        * Netty
        * OpenAPI tools (validation, etc.)
        * async clients for making database queries, HTTP requests, etc.
        * and more
    * the `Future` type
        * API methods often return it
        * flatMap operation on it is key
        * Vert.x v4 requires dealing with them
            * v3 API methods often accepted a `null`-returning callback lambda, and returned `null` themselves
            * `null`, `null` everywhere
            * deprecated now
        * however, long flatMap chains can be difficult to read
            * give names to intermediate results
            * practice with Java standard library types: `Stream`, `Optional`

---

## Implementation: IP address checking

```java
/**
 * Checks if an IP address belongs to a network.
 *
 * @param ipAddress The IP address
 * @param networkSubnets The collection of subnets that defines a network
 * @return Whether the IP address belongs to any subnet in the collection
 */
static boolean isOnNetwork(Ip4 ipAddress, Cidr4Trie<String> networkSubnets) {

    return networkSubnets.shortestPrefixOfValue(new Cidr4(ipAddress), true) != null;
}
```

???

* third-party Java library that implements a CIDR radix tree
* seems to be common use case of trie, so hopeful that impls in other languages are available

---

## Implementation: access cookie checking

* Duplication of cookie decryption algorithm used by content provider site

    * Duplication of secret key

???

* this impl not ideal due to the duplication (DRY)

---

## Thinking ahead: Auth 2.0

Essentially: a more general data model / higher level of abstraction

* **access service:** not just cookies anymore

* **probe service:** not just Image resources anymore

* service**_s_** (plural): one-to-many relationships are possible

???

* the HTTP Cookie header is no longer the required "authorizing aspect" of the client
    * new possibilities: IP address, User-Agent, etc.
    * no more unnecessary access cookies for IP address-based auth, for example
* info.json is no longer the carrier of a resource's auth requirements: "probe service"
* can have a one-to-many relationship between a content resource and auth services

---

## Thinking ahead: Auth 2.0 and Hauth

I have lots of questions

???

* tbh: my understanding of Auth 2.0 is still fuzzy
* however, I believe that much of Hauth can be generalized / reused in a 2.0 impl
    * "campus network user" impl can be simplified (no access cookie needed)
    * 2.0 seems to more clearly suggest a general implementation for "bespoke research site user" impl

---

## Reach out to me!

<a href="mailto:mmatney@library.ucla.edu">mmatney@library.ucla.edu</a>

Code4Lib, UC Tech, IIIF Slack: @markmatney

![QR encoding of slides URL][slides]

  [slides]: ./slides-qr-code.svg

<br>

_Created with:_ https://remarkjs.com

???

* feel free to reach out to me with implementation questions about Auth 1.0
* I'm on Slack
* also: highly recommend RemarkJS if you like LaTeX
* _switch screenshare to speaker notes (so meta)_
