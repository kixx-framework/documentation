# Publishing API

The Publishing API allows clients to upload templates, page metadata, includes content, and static assets to a remote Kixx instance.

Base path: `/publishing-api/v1`

## Media Type

The Publishing API adheres to the JSON:API specification. There are some raw upload endpoints which accept other content types, but most endpoints require and emit:

```
Content-Type: application/vnd.api+json
```

A request to a JSON:API endpoint that sends any other `Content-Type` is rejected with `415 Unsupported Media Type`. The media type comparison is exact: optional parameters on the header (e.g. `; charset=utf-8`) are not accepted by the content-type assertion, so send the bare media type.

Raw upload endpoints (templates, includes, static assets) use other, appropriate media types; see those sections below.

## Resource Documents

Request bodies for JSON:API write endpoints are single-resource documents:

```json
{
    "data": {
        "type": "AdminUser",
        "attributes": { "...": "..." }
    }
}
```

The parser enforces:

- `data` must be an object → otherwise `400 BadRequestError`.
- `data.type` must be a non-empty string → otherwise `400 BadRequestError`.
- `data.type` must equal the type the endpoint expects → otherwise `409 ConflictError` with code `JsonApiResourceTypeMismatch`.
- `data.attributes` must be an object → otherwise `400 BadRequestError`.

`data.id` is accepted but ignored by endpoints where ids are derived server-side.

## Response Documents

Successful writes respond with a single-resource document:

```json
{
    "data": {
        "type": "PublishingApiToken",
        "id": "…",
        "attributes": { "...": "..." },
        "meta": { "...": "..." }
    }
}
```

`id` and `meta` are present only when the endpoint sets them.

## Error Documents

Errors are serialized into the JSON:API error format:

```json
{
    "errors": [
        {
            "status": "422",
            "code": "ValidationError",
            "title": "ValidationError",
            "detail": "Password must be at least 16 characters",
            "source": "password"
        }
    ]
}
```

Field reference:

- `status` — string form of the HTTP status code.
- `code` — stable application error code. For framework HTTP errors this is the error's `code` (e.g. `NewUserConflictError`, `InvalidInvite`); for unexpected errors it is `INTERNAL_SERVER_ERROR`.
- `title` — the error class name (e.g. `ValidationError`, `ConflictError`), or `InternalServerError` for unexpected errors.
- `detail` — a client-safe message. Internal/unexpected error messages are never exposed; they are replaced with `"Internal server error"`.
- `source` — present on validation field errors; identifies the offending field.

A single `ValidationError` can expand into multiple `errors` entries, one per invalid field, all sharing the same `status`/`code`/`title`. SDK clients should treat `errors` as an array and surface each `detail`/`source` pair.

A `405 Method Not Allowed` response additionally carries an `Allow` header listing the supported methods for the route.

## Authentication and Authorization

Every Publishing API request requires a publishing bearer token:

```
Authorization: Bearer kxpat_<...>
```

- missing/empty tokens are rejected with `401 UnauthenticatedError`.
- unknown tokens are rejected with `401 UnauthenticatedError`.
- expired or revoked tokens are rejected with with `403` code `PublishingApiTokenInactive`.

Each endpoint performs a per-request permission check via the token's grants. Insufficient permissions on a token yields `403` code `PublishingApiTokenForbidden`.

## Build ID Header

```
Kixx-Build-Id: <build-id>
```

The build id namespaces uploaded content so a new deployment can be staged without disturbing the live build. The header behaves **differently** per endpoint:

- **Templates:** the Kixx-Build-Id header is **required**, and it must be a build id **other than the current (live) build**. Template writes always target a staged, non-current build. When bootstrapping a new site for the first time the current (live) build will be null, but templates must still target a staged build.
- **Page metadata and includes:** the header is **optional**. When omitted, the write targets the current (live) build. When the deployment has no current build configured and no header is supplied, the write fails with `409` code `CurrentBuildIdRequired`.

> Live-build note: an include written to the current build only becomes visible
> after the owning page is re-`PUT` (page metadata) with a bumped `version`,
> because include content is cached by page version.

## Pathname / Filepath Safety

When uploading templatese or content the URL pathnames are validated before reaching storage. A path is rejected with `400 BadRequestError` ("Invalid pathname …") when it:

- contains `..` or `//`,
- has any segment beginning with `.` (dotfiles),
- contains any character outside `[a-z0-9_.-]` (case-insensitive).

## 1. Put a Template

```
PUT /publishing-api/v1/templates/base/*filepath       (kind: base)
PUT /publishing-api/v1/templates/pages/*filepath       (kind: page)
PUT /publishing-api/v1/templates/partials/*filepath    (kind: partial)
```

Uploads a single template source file into a staged build namespace.

### Headers

```
Authorization: Bearer kxpat_<...>
Content-Type: text/plain        (or text/html)
Kixx-Build-Id: <non-current-build-id>
```

`Content-Type` must be `text/plain`; any other type will result in an HTTP `415` response.

**Request body** — the raw template source text (non-empty). The body is read as text, not JSON.

**Permission decision performed**

```
action:   urn:kixx:publishing:template:put
resource: urn:kixx:publishing:template
```

**Success — `200 OK`** (JSON:API `Template`):

```json
{
    "data": {
        "type": "Template",
        "id": "<filepath>",
        "attributes": {
            "kind": "base",
            "filepath": "<filepath>",
            "buildId": "<build-id>"
        }
    }
}
```

`filepath` in the response is the logical filepath Hyperview stored the template under (the wildcard segments joined by `/`, with no leading slash).

**Errors**

| Status | code | Cause |
|---|---|---|
| 415 | `UnsupportedMediaTypeError` | Body is not text/plain or text/html. |
| 400 | `TemplateFilepathRequired` | Empty wildcard filepath. |
| 400 | (invalid pathname) | Filepath fails path-safety validation. |
| 400 | `TemplateSourceRequired` | Empty request body. |
| 400 | `BuildIdRequired` | `Kixx-Build-Id` header missing. |
| 409 | `CurrentBuildIdRequired` | Deployment has no current build configured. |
| 409 | `CurrentBuildWriteConflict` | `Kixx-Build-Id` equals the current build. |
| 401/403 | (auth) | See Authentication above. |

## 2. Put a Page Metadata File

```
PUT /publishing-api/v1/pages/*pathname
```

Writes a page's metadata document (`page.json`) for the given page pathname.

**Headers**

```
Authorization: Bearer kxpat_<...>
Content-Type: application/vnd.api+json
Kixx-Build-Id: <build-id>          (optional; defaults to current build)
```

**Request body** — JSON:API resource of type `PageMetadata`. The `attributes` object is an arbitrary bag of JSON that becomes the page's `page.json`, with one required field:

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `version` | string | yes | Page content version used by live-build cache keys. |
| (any others) | any | no | Arbitrary page metadata; passed through unchanged. |

```json
{
    "data": {
        "type": "PageMetadata",
        "attributes": {
            "version": "2026-06-28-1",
            "page": { "title": "Home" },
            "baseTemplate": "website.html"
        }
    }
}
```

**Permission decision performed**

```
action:   urn:kixx:publishing:page-metadata:put
resource: urn:kixx:publishing:page-metadata:<pathname>
```

(`pathname` here is the normalized, leading-slash form, e.g. `/blog/hello`.)

**Success — `200 OK`** (JSON:API `PageMetadata`):

```json
{
    "data": {
        "type": "PageMetadata",
        "id": "/blog/hello",
        "attributes": {
            "version": "2026-06-28-1",
            "page": { "title": "Home" },
            "baseTemplate": "website.html"
        },
        "meta": { "buildId": "<effective-build-id>" }
    }
}
```

`attributes` echoes the full metadata bag. `meta.buildId` reports the build the write actually targeted (the supplied header, or the current build).

**Errors**

| Status | code | Cause |
|---|---|---|
| 415 | `UnsupportedMediaTypeError` | Body is not `application/vnd.api+json`. |
| 409 | `JsonApiResourceTypeMismatch` | `data.type` is not `PageMetadata`. |
| 400 | `PagePathnameRequired` | Empty wildcard pathname. |
| 400 | (invalid pathname) | Pathname fails path-safety validation. |
| 422 | `ValidationError` | Attributes not an object, or missing `version`. |
| 409 | `CurrentBuildIdRequired` | No header and no current build configured. |
| 401/403 | (auth) | See Authentication above. |

## 3. Put an Included Content File

```
PUT /publishing-api/v1/includes/*filepath
```

Writes an include content file (referenced from a page's `includes`) for the page directory it lives in. The wildcard `*filepath` is split into the owning page pathname (all but the last segment) and the include filename (last segment).

NOTE: An include file will not be available as part of the page rendering context until it is also included in the page metadata `includes` list.

**Headers**

```
Authorization: Bearer kxpat_<...>
Content-Type: text/*               (any text/* media type)
Kixx-Build-Id: <build-id>          (optional; defaults to current build)
```

`Content-Type` must begin with `text/`; otherwise `415` (`Accept: text/*`).

**Request body** — the raw include source text (non-empty). Read as text, not JSON.

**Path splitting example** — `PUT /publishing-api/v1/includes/blog/hello/intro.md`:

- `filepath` → `blog/hello/intro.md`
- `pathname` → `/blog/hello`
- `filename` → `intro.md`

A single-segment filepath (e.g. `intro.md`) resolves to `pathname` `/`.

**Permission decision performed**

```
action:   urn:kixx:publishing:include:put
resource: urn:kixx:publishing:include:<filepath>
```

**Success — `200 OK`** (JSON:API `Include`):

```json
{
    "data": {
        "type": "Include",
        "id": "blog/hello/intro.md",
        "attributes": {
            "pathname": "/blog/hello",
            "filename": "intro.md",
            "buildId": "<effective-build-id>"
        }
    }
}
```

**Errors**

| Status | code | Cause |
|---|---|---|
| 415 | `UnsupportedMediaTypeError` | Body is not a `text/*` type. |
| 400 | `IncludeFilepathRequired` | Empty wildcard filepath. |
| 400 | (invalid pathname) | Filepath fails path-safety validation. |
| 400 | `IncludeSourceRequired` | Empty request body. |
| 409 | `CurrentBuildIdRequired` | No header and no current build configured. |
| 401/403 | (auth) | See Authentication above. |
