# Packaging on the Web

This document describes an approach for creating packages of files for use on the web. The approach is to package them using a new `multipart/package` media type and a `+package` structured syntax. To access packages related to other files on the web, clients that understand packages of files look for a `Link` header or (in HTML documents) a `<link>` element with a new link relation of `package`. Other formats may define format-specific mechanisms for locating related packages.

**This is an unreviewed draft by Jeni Tennison and does not represent official TAG or W3C opinion.**

## Requirements

There are two main requirements for packages on the web:

  * efficient delivery of related content on the web
  * easy distribution of self-contained content

There is also a final cross-cutting requirement: that the solution can be easily and backwards-compatibly deployed on the web.
  
### Efficient Delivery

If a user visits `http://www.bbc.co.uk/` they will need to download about 160 files to view the page in its entirety. The HTML page they download at `http://www.bbc.co.uk/` contains references to stylesheets, scripts, images and other files, each of which may contain references to further files themselves.

Delivering a package of these files could be more efficient than delivering individual files. Downloading each file has a connection overhead which is particularly impactful on low-bandwidth mobile devices and on secure connections.

The browser can't work out which additional files to download until it receives the HTML page, but the server could plausibly deliver a package that contains all the required files through a single connection. This would have to work in a backwards compatible way for both older browsers interacting with package-aware servers, and for package-aware clients working with older servers.

> *Note: Efficient delivery is the aim of [pipelining in HTTP 1.1](http://www.w3.org/Protocols/rfc2616/rfc2616-sec8.html) (and [HTTPbis](http://tools.ietf.org/html/draft-ietf-httpbis-p1-messaging-25#section-6.3.2)) and [multiplexing in HTTP 2.0](http://tools.ietf.org/html/draft-ietf-httpbis-http2-09#section-2.2). These enable multiple requests to be passed, and responded to, over the same persistent connection. But they do not enable the server to deliver content for predicted requests.*

> *Note: The `rel=prefetch` link relation prompts the browser to access additional HTML pages that have not yet been requested by the user, but again this is a different facility; there is no such think as prefetching stylesheets, scripts or images as these are already required for the page by the time the browser knows about them.*

> *Note: given that browsers can typically have more than one connection open to a website, and download files in parallel, is there an argument for supporting having multiple packages associated with a given page?*


### Self-Contained Content

It is sometimes useful to distribute self-contained packages of material. Examples are [Packaged Web Apps](http://www.w3.org/TR/widgets/) or packages of CSV files used within the [Simple Data Format](http://dataprotocols.org/simple-data-format/) or the [DataSet Publishing Language](https://developers.google.com/public-data/). These packages typically contain a manifest file, in a machine readable format (text, JSON or XML), that contains further details and metadata about the files that the package contains.

## Packaging Format

> *Note: For rationale for selecting this above other packaging formats, see 'Rejected Approaches' below.*

The TAG recommends using [multipart media types](http://tools.ietf.org/html/rfc2046#section-5.1) for packaging materials on the web. Specifically we recommend the registration of a new `multipart/package` media type and the registration of a new `+package` structured syntax suffix, per [RFC 6838](http://tools.ietf.org/html/rfc6838#section-6).

Multipart media types are defined in [RFC 2046](http://tools.ietf.org/html/rfc2046#section-5.1). They can contain one or more *body parts*, each comprising:

  * a *boundary*
  * a number of *header fields*
  * an empty line
  * the *content* of the file

> *Ed note: From what I can tell it's possible for the header fields within a multipart response to be anything so long as it follows the normal `Header: Value` syntax used in MIME and HTTP. So this could be a very flexible mechanism for adding metadata.*

The `multipart/mixed` media type places no constraints on which header fields are specified within the multipart file (see the definition of `MIME-part-headers` in [RFC 2045](http://tools.ietf.org/html/rfc2045#section-3)).

The only difference between `multipart/mixed` and `multipart/package` is that the header fields in every body part includes the `Content-Location` header. Further, the values of the `Content-Location` header must all be [absolute-path-relative URLs](http://url.spec.whatwg.org/#concept-absolute-path-relative-url) or [path-relative URLs](http://url.spec.whatwg.org/#concept-path-relative-url). (See Security Considerations later.)

### Fragment Identifiers for Packages

A basic fragment identifier scheme for the `multipart/package` media type is of the form `file=url` where *url* is resolved against the base URL of the package, the part before the fragment in *url* is used to identify a file within the package using the `Content-Location` part header, and any fragment in *url* is used to identify a fragment within that file according to the media type for that file (as given in the `Content-Type` header).

For example, the URL

    http://example.org/path/to/package.pack#file=/home.html%23section1
    
refers to the file within `http://example.org/path/to/package.pack` whose `Content-Location` is

    http://example.org/home.html

and more specifically the element with the `id` `section1` within that file. This should be the same as `http://example.org/home.html#section1`.

In general, links should be made directly to files on the web rather than to files within packages. The particular package(s) that a file appears in is an ephemeral phenomenon and not suitable for inclusion in a URL.

### `+package` Structured Suffix

The `+package` structured suffix should be used on other multipart media types that are used for more specialised packages. This is particularly useful for package formats that must contain manifest files in particular formats; these should use the `+package` structured suffix in their media type.

For example, a `multipart/widget+package` media type could specify that a web application package must contain an [`config.xml` configuration document](http://www.w3.org/TR/widgets/#configuration-document) using the `http://www.w3.org/ns/widgets` XML vocabulary as the first file within the package, and that all other files in the package must be listed within this configuration file or ignored.

Clients can treat any file with an unrecognised `+package` media type as if it were a `multipart/package` file.

## Requesting a Package

Packages live on the web just like any other file. Thus it is perfectly possible to request a package directly. For example:

    GET /path/to/package.pack HTTP/1.1
    Accept: multipart/package,multipart/*,*/*

should result in a response like:

    HTTP/1.1 200 OK
    Content-Type: multipart/package;boundary=package-boundary
    
    ... package content ...

This satisfies the second of the requirements described above, namely the easy distribution of self-contained content. Note that the `boundary` parameter is required for multipart media types as defined in RFC 2046.

To locate a package of representations of related resources, to support the efficient delivery of scripts, stylesheets, images and so on over the web, we recommend the use of a new `package` link relation. This can be used within a `<link>` header in an HTML document:

    <link rel="package" href="/path/to/package.pack">

When the package is not HTML-based (for example if it is defined through a metadata file defined in JSON or XML), the `package` link relation can be used within a `Link` header:

    Link: </path/to/package.pack>; rel="package"

### Processing Packaged Content

Clients that receive packaged content should unpackage by splitting the package on the boundary indicated in the media type. If there was no `Content-Type` header or no `boundary` parameter on the given content type then clients may recover by inferring the boundary from the content of the packaged content.

> *Note: I guess that the `boundary` parameter is required because there were implementations at the time of standardisation that didn't start with `--boundary`. Now it's a real pain as it means multipart files can't be self-contained.*

The `Content-Location`s associated with each packaged file must be resolved relative to the location of the package. If this results in a location that has a different origin from the package, the file must be ignored.

For performance, it is good practice for the first file in the package to be the "root" of the package, referencing all the other files, but this cannot be relied upon by the client as the same package may be used for multiple resources.

Other files may be cached for later use by the client, with headers set as appropriate based  on those provided within the package and within the response to the original request.

## Security Implications

When used with the `Package` header, the goal of a package is to populate a client's cache and prevent it from making additional unnecessary requests. Content from the cache will run with a base URL supplied within the package. If the `Content-Location`s given for the files in a package weren't restricted to the domain that the package is hosted on, `evil.com` could deliver content that it claimed came from `bank.com` and that would then be interpreted as if it did, indeed, come from `bank.com` but that ran scripts inserted by `evil.com`. 

For the same reason, publishers should be careful about the packages that they provide on their site, in just the same way as they should avoid hosting untrusted HTML.

## Rejected Approaches

Other approaches to packaging have been used and were considered by the TAG.

### Zip as a Packaging Format

The TAG discussed the use of zipped files as a packaging format. The main problem with using zips is that the *central directory record*, which lists the valid files within the zip archive, appears at the *end* of the zip. Implementations therefore need to wait until the whole zip is downloaded before the files within it can be read. This makes it unsuitable for a packaging format for efficient delivery of content on the web (the first of the requirements described above).

A secondary problem with zip as a packaging format is that while there are mechanisms for supplying additional information about individual files within the package (through *extra fields*), they are not sufficient for extended metadata. Each extra field is a 2-byte ID code with a 2-byte value. The list of valid core and extended ID codes are provided within section 4.5 and 4.6 of the [zip definition](http://www.pkware.com/documents/casestudies/APPNOTE.TXT). The file header within the zip, which includes these extra fields, must not exceed 64k in size.

These limitations have resulted in people who use zip as a packaging format providing separate manifest files within the zip.

### Other Packaging Formats

#### Mozilla Archive Format

The [Mozilla Archive Format](http://maf.mozdev.org/maff-specification.html) is a zip-based packaging format for web content which uses an RDF/XML manifest file within the zip to provide additional information about the content. This approach has the drawbacks described in the previous section, particularly lack of streamability.

#### MHTML

[RFC 2557](http://tools.ietf.org/html/rfc2557) defines MIME Encapsulation of Aggregate Documents, such as HTML (MHTML). This uses the `multipart/related` media type, with the first file in the package being the packaged HTML document and the remainder being related resources.

This is not a suitable general format for publishing packages on the web as it is designed around an HTML page being the primary starting point for a package, which is not true in all circumstances.

#### Webarchive

The [Webarchive](http://en.wikipedia.org/wiki/Webarchive) format uses `application/x-webarchive` as a media type. It is a proprietary format defined by Apple and used within Safari. There is very little information available about its internal structure.

#### WARC

The [WARC](http://www.digitalpreservation.gov/formats/fdd/fdd000236.shtml) format is used for archiving web content. Although it provides for packaging, and metadata for the files within the package, it is designed for archiving and is fairly heavyweight for the packages that are under discussion here, requiring `WARC-Record-ID`, `Content-Length`, `WARC-Date` and `WARC-Type` headers.

### Package Requests

An approach to requesting packages that we considered would be to include a new `Package: true` header in HTTP requests for normal files on a web server. Servers that understand the `Package` header could then respond with a new `2XX Packaged Content` success response whose body is a package that includes a representation of the requested resource, along with representations of any other related resources.

For example, a client that understood packaging would send a request like:

    GET /home.html HTTP/1.1
    Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
    Package: true

The `Package` header with the value `true` would indicate that the server should attempt to respond with a package that includes the requested resource. If the server is a legacy server that does not understand the `Package` header, or if the server understands the `Package` header but does not have a suitable package with which it can respond, it will respond as normal to this request:

    HTTP/1.1 200 OK
    Content-Type: text/html
    
    ... content of /home.html ...

If the server understands the `Package` header and can respond with a package that contains the requested representation (`/home.html`) then it should respond with a `2XX Packaged Content` response. The `Content-Location` header in this response would indicate the location of the package:

    HTTP/1.1 2XX Packaged Content
    Content-Type: multipart/package;boundary=package-boundary
    Content-Location: /path/to/package.pack

    ... content of /path/to/package.pack ...

A `2XX Packaged Content` response would indicate that the server is responding with a package that includes the same representation for the requested resource as would have been provided, with a `200 OK` response, if the `Package` header had not been present in the request.

The problem with this approach is that it requires some fairly large changes to HTTP: a new HTTP header and a new HTTP status code. These are complicated to implement both in terms of specification and in terms of getting servers and clients to support them. New status codes in particular are difficult to plug in to popular web servers such as Apache. Using a non-standard status code also requires configuration access to servers, which isn't possible in many publishing environments.

### Specialising URLs

The TAG [investigated the use of a special URL syntax](https://gist.github.com/wycats/220039304b053b3eedd0) that would enable package-aware clients to work with packages whilst legacy clients and servers work with individual files. This approach is designed to meet the requirement that someone could use it on a file-system-based web server without access to any configuration options. In other words, it does not require servers to be package aware.

> **Note: This requirement also entails using a self-contained packaging format. The multipart format described above is not self-contained because it requires the `boundary` parameter to be set via a `Content-Type` header. Zip packages or multipart packages nested inside `message/http` documents are alternative self-contained packaging formats.**

For example, we explored using:

    http://example.com/path/to/package.pack!/home.html#section1
    
to indicate the anchor `section1` within the file `home.html` within the package `/path/to/package.pack`. The separator `!/` is a proposed unique separator between the package location and the location of the target file within the package.

If someone wanted to provide packages for their files, they would structure their URL space so that it looked like:

    path/
      to/
        package.pack
        package.pack!/
          home.html

A package-aware client would recognise that the URL `http://example.com/path/to/package.pack!/home.html#section1` contained the package separator `!/`. Instead of directly requesting the file `http://example.com/path/to/package.pack!/home.html` as a legacy client would, it would request the file `http://example.com/path/to/package.pack`, unpack the package, use the contents of the package to populate its cache, and then navigate to `http://example.com/path/to/package.pack!/home.html`, which would then be within the cached content.

The separator `!/` is designed such that it is unlikely to appear in existing URLs [TODO: some analysis on whether this is actually the case]. It is also designed to enable relative links to work. If there is a link within `home.html` to `faq.html` in the same package, you would want to write within the page simply:

    <a href="faq.html">FAQ</a>

With a base URL of `http://example.com/path/to/package.pack!/home.html#section1` such a link would resolve to `http://example.com/path/to/package.pack!/faq.html`. Similarly, links that started with `.` or `..` would continue to resolve as expected; the package works exactly as a directory.

This approach could be effectively polyfilled using [Service Workers](https://github.com/slightlyoff/ServiceWorker/blob/master/explainer.md). The Service Worker would intercept two types of requests:

  1. requests that include `!/` would be mapped into requests for the package; the resulting package would be used to populate a content cache containing the unpacked package
  2. further requests for pages that are controlled by the Service Worker would be fulfilled from the populated content cache where packaged content has been provided

Implementation through Service Worker enables sites to use this packaging method without any cross-site standardisation effort.

The biggest architectural problem with standardising this approach is that it places additional constraints on URL spaces, at least for items for which a package should be downloaded. As detailed in the [Internet Draft Standardising Structure in URIs](http://tools.ietf.org/html/draft-ietf-appsawg-uri-get-off-my-lawn-00), there are risks when defining new standard internal structures within URLs:

  * **collisions**: the suggested convention of `!/` may clash with URL conventions used on other systems that have different best practices for URL structures
  * **dilution**: the arrangement of files into packages is ephemeral information and does not reflect the semantic content of the files; it is bad practice to include ephemeral information in URLs as it makes those URLs likely to change, and therefore links to break
  * **brittleness**: baking in a particular new URL structure into the web is a far reaching change that will be hard to change in the future
  * **operational difficulty**: creating URLs containing `!/` may be difficult in some systems, for example where it is hard to create directories that contain the `!` character
  * **client assumptions**: there may be existing URLs that contain the package delimiter (eg `!/`) that would break with new package-aware clients

The issues of dilution and operational difficulty are particularly apparent when considering a file that should appear in multiple packages. The person managing the server would have to ensure it's duplicated whenever it's updated; those referencing the file would have to choose which instance of the file to reference depending on which other files should be packaged with it.

### Content Negotiation

The TAG explored the use of content negotiation to retrieve a package of resources. In this scenario, a client that understood packages would include `multipart/package` as the most-favoured type of response:

    GET /home.html HTTP/1.1
    Accept: multipart/package,text/html;q=0.95,application/xhtml+xml;q=0.95,application/xml;q=0.9,image/webp,*/*;q=0.8

A server that had a package containing `/home.html` would respond with that package:

    HTTP/1.1 200 OK
    Content-Type: multipart/package;boundary=boundary-in-home-package.pack
    Content-Location: /home-package.pack

There are three potential problems with this approach.

First, a package that contains `/home.html` is arguably not a representation of the resource `/home.html`, only a container for such a representation.

> *Note: It's not clear whether the fact that there's a mismatch in semantics actually has any implementation impact.*

Second, the server would still need to use the rest of the `Accept` header to determine what to include within the package, or indeed whether a package can be created at all for the resource. For example, if the request had an `Accept` header of:

    Accept: multipart/package,text/json;q=0.9

then we would like the server to respond with a package that contained the `text/json` representation of the requested resource, or to give a `406 Not Acceptable` response if there was no such package. This ability to dig into the remaining part of the `Accept` header to determine a response would require revising the way in which the `Accept` header works, which we can't do.

Third, there would be no mechanism to differentiate between requesting a package directly and requesting a package that contains a packaged resource. For example, say that CSV and metadata were packaged together into `multipart/package` files like `http://example.com/data.pack`. It would not be clear from a request like:

    GET /data.pack HTTP/1.1
    Accept: multipart/package

whether the request was directly for `/data.pack` or for a package that contained `/data.pack`.