- Start Date: 2020-07-22
- RFC PR: [#36](https://github.com/inveniosoftware/rfcs/pull/36)
- Authors: Nicola Tarocco
- State: DRAFT

# HTTP Cache response headers when fetching records using REST APIs

## Summary

When fetching a record via REST APIs using the endpoint provided by `invenio-records-rest`, Invenio will return the response and a set of HTTP cache headers. In particular, it will return the headers `ETag` and `Last-Modified` that help the client to correctly cache the content.
In the specific case of web browsers as client fetching a record, for example via `AJAX` requests, the absence of `Cache-Control` header is basically allowing browsers to decide what client cache strategy to apply, which currently causes issues.

For a simple and complete HTTP cache documentation, see [here](https://web.dev/http-cache/).

## Motivation

`invenio-records-rest` takes advantage of `ETag` and `Last-Modified` as cache strategy: the expected behavior is that the client will **always** perform the HTTP request, and the response will be `304 Not Modified`, with no content, when the client has already the same version of the record, or `200 OK`, with the record content, in all other cases.
Unfortunately, since Invenio is not setting the `Cache-Control` header in the response, browsers will decide on their own if the request should be performed or not, defeating the cache strategy. In most cases, what is happening is that browsers are actually caching the response and not performing subsequent requests.
Invenio should instead have control on the client caching strategy by providing the correct set of cache headers in the response.

For more information, see [here](https://www.mnot.net/blog/2017/03/16/browser-caching#heuristic-freshness), in particular:

> If a response doesn’t have explicit freshness information like Expires or Cache-Control: max-age, HTTP still allows it to be cached using what’s called heuristic freshness.
> ...
> All tested browser caches apply heuristic freshness to 200 OK.

## Detailed design

The correct set of HTTP cache response headers should always include `Cache-Control` and also `ETag` and/or `Last-Modified`. This will allow the server to control how browsers will cache responses.
`invenio-records-rest` responses will be changed to include the `Cache-Control: no-cache` response header.

## Example

Browser request:

```
GET https://localhost:5000/api/records/84756

Host: localhost:5000
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:77.0) Gecko/20100101 Firefox/77.0
Accept: application/json, text/plain, */*
Accept-Encoding: gzip, deflate, br
...
```

Expected Invenio response:

```
HTTP/1.0 200 OK
Content-Type: application/json
Content-Length: 6128
Cache-Control: no-cache
ETag: "1"
Last-Modified: Fri, 10 Jul 2020 16:19:24 GMT
Link: <https://localhost:5000/api/records/84756>; rel="self"
Vary: Origin
...
```

## Drawbacks

No drawbacks found at this moment.

## Alternatives

No alternative found when using `ETag` or `Last-Modified`.
