# Packaging on the Web

This TAG activity aims to provide mechanisms that enable better use of packages on the web, for a variety of reasons:

  * as a tool for improving performance
  * as a mechanism for distributing modular components
  * as a way of providing both data and metadata in a single file

**The draft specification of our recommended approach is now available at [http://w3ctag.github.io/packaging-on-the-web/](http://w3ctag.github.io/packaging-on-the-web/).**

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