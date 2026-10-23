## HTTP caching: Cache-Control

The server tells the client (browser, proxy, CDN) whether and how long a response can be reused, with the `Cache-Control` response header:

```
Cache-Control: max-age=3600
```

- **`max-age=N`**: the response is fresh for N seconds. During that time, the client reuses it **without any request** to the server.
- **`no-cache`**: can be stored, but must be revalidated with the server before each reuse (see conditional requests below). Misleading name: it doesn't mean "don't cache".
- **`no-store`**: never stored anywhere. For sensitive data (banking pages, personal data).
- **`private`** / **`public`**: only the browser can store it (user-specific response), or shared caches too (proxy, CDN).
- **`immutable`**: the content never changes, no revalidation even on reload. Typical for hashed static files (`main.3f9a2c.js`), with a long `max-age`.

Typical setup for an Angular app: `index.html` with `no-cache` (always the latest version, which references the new hashed bundles), hashed bundles with `max-age=31536000, immutable`.

## Conditional requests: ETag and Last-Modified

Once a cached response is stale (or with `no-cache`), the client doesn't download it again blindly: it asks the server whether it changed. If not, the server answers **`304 Not Modified`** with no body, and the client reuses its copy. Saves bandwidth, not the round trip.

Two mechanisms, the server sends a validator with the response and the client sends it back:

| Response header (server)                       | Request header (client)          | Compares                     |
| ---------------------------------------------- | -------------------------------- | ---------------------------- |
| `ETag: "a1b2c3"` (version id, e.g. a hash)     | `If-None-Match: "a1b2c3"`        | the ETag, exact match        |
| `Last-Modified: Tue, 21 Oct 2026 10:00:00 GMT` | `If-Modified-Since: <same date>` | the date, 1-second precision |

```
GET /api/books/42                    →  200 OK
                                        ETag: "v7"
                                        Cache-Control: no-cache
                                        { ...body... }

GET /api/books/42                    →  304 Not Modified    (unchanged, no body)
If-None-Match: "v7"

GET /api/books/42                    →  200 OK               (changed: new body, new ETag)
If-None-Match: "v7"                     ETag: "v8"
```

- `ETag` is more precise (no date granularity issue, works for content not tied to a file date) and takes precedence when both are sent.
- In Spring: `ShallowEtagHeaderFilter` computes an ETag from the response body hash (saves bandwidth, but the server still builds the response), or set it explicitly with `ResponseEntity.ok().eTag("\"v7\"").body(book)` and check `WebRequest.checkNotModified(etag)` to skip the work.
- Related, for writes: `If-Match: "v7"` on a `PUT` makes the server reject the update with `412 Precondition Failed` if the resource changed meanwhile (optimistic locking, like Hibernate's `@Version`).
