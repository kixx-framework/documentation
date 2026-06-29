# Admin API

The Admin API manages admin user account creation (invite acceptance) and the minting of Publishing API tokens. It is **not** protected by the admin session cookie — each endpoint authenticates with its own scheme (see [Authentication](#authentication)).

Base path: `/admin-api/v1`

Two endpoints are available, both `POST`:

| Endpoint | Purpose | Authentication |
|---|---|---|
| `POST /admin-api/v1/users/invite` | Redeem an invite token to create an admin account | Bearer invite token |
| `POST /admin-api/v1/publishing-api-tokens` | Mint a Publishing API token for an admin | HTTP Basic (admin email + password) |

## Media Type

Both endpoints require a JSON:API request body and emit JSON:API documents (both success and error) with:

```
Content-Type: application/vnd.api+json
```

A request that sends any other media type is rejected with `415 Unsupported Media Type` (code `UNSUPPORTED_MEDIA_TYPE_ERROR`). The content-type check runs before authentication, so a wrong media type is rejected before any credential is examined.

Before comparison the `Content-Type` is normalized: header parameters (e.g. `; charset=utf-8`) are stripped and the media type is lowercased. So `application/vnd.api+json; charset=utf-8` is accepted, but a different media type is not.

## Resource Documents

Request bodies are JSON:API single-resource documents:

```json
{
    "data": {
        "type": "AdminUser",
        "attributes": { "...": "..." }
    }
}
```

The parser enforces:

- the request body and its `data` member must both be objects → otherwise `400 BadRequestError`.
- `data.type` must be a non-empty string → otherwise `400 BadRequestError`.
- `data.type` must equal the type the endpoint expects (`AdminUser` or `PublishingApiToken`) → otherwise `409 ConflictError` with code `JsonApiResourceTypeMismatch`.
- `data.attributes` must be an object → otherwise `400 BadRequestError`.

A request body that is not valid JSON is rejected with `400 BadRequestError` ("Invalid JSON in request body") before envelope validation runs.

`data.id` is accepted but ignored; both endpoints derive the resource `id` server-side.

## Response Documents

Successful writes respond `201 Created` with a JSON:API single-resource document:

```json
{
    "data": {
        "type": "AdminUser",
        "id": "…",
        "attributes": { "...": "..." }
    }
}
```

Neither endpoint sets a `meta` object; every returned field lives under `attributes`.

## Error Documents

Errors are serialized into the JSON:API error format:

```json
{
    "errors": [
        {
            "status": "422",
            "code": "VALIDATION_ERROR",
            "title": "ValidationError",
            "detail": "Password must be at least 16 characters",
            "source": "password"
        }
    ]
}
```

Field reference:

- `status` — string form of the HTTP status code.
- `code` — stable application error code. For framework HTTP errors this is the error's `code`: either the error class's default code (e.g. `VALIDATION_ERROR`, `BAD_REQUEST_ERROR`, `UNSUPPORTED_MEDIA_TYPE_ERROR`) or a custom code set at the throw site (e.g. `JsonApiResourceTypeMismatch`, `InvalidInvite`, `InvalidCredentials`, `NewUserConflictError`). For unexpected errors it is `INTERNAL_SERVER_ERROR`.
- `title` — the error class name (e.g. `ValidationError`, `ConflictError`), or `InternalServerError` for unexpected errors.
- `detail` — a client-safe message. Internal/unexpected error messages are never exposed; they are replaced with `"Internal server error"`.
- `source` — present on validation field errors; identifies the offending field.

A single `ValidationError` can expand into multiple `errors` entries, one per invalid field, all sharing the same `status`/`code`/`title`. SDK clients should treat `errors` as an array and surface each `detail`/`source` pair.

A `405 Method Not Allowed` response additionally carries an `Allow` header listing the supported methods for the route. Both routes allow only `POST`.

## Authentication

The Admin API does not use the admin session cookie. Each endpoint authenticates independently, and in both cases the JSON:API `Content-Type` is validated (415) before the credential is examined:

- **Accept an invite** uses a **bearer invite token**: `Authorization: Bearer <invite-token>`. See [Accept an Admin User Invite](#1-accept-an-admin-user-invite).
- **Create a token** uses **HTTP Basic** admin credentials: `Authorization: Basic <base64>`. See [Create a Publishing API Token](#2-create-a-publishing-api-token).

## 1. Accept an Admin User Invite

```
POST /admin-api/v1/users/invite
```

The trailing slash is optional (`/users/invite` and `/users/invite/` both match).

Creates an admin user account by redeeming an invite token. This complements the browser HTML invite-acceptance flow so CLI/SDK clients can register without a browser. It does **not** establish a session; it only creates the account.

### Authentication — bearer invite token

```
Authorization: Bearer <invite-token>
```

The token is either:

- A stored single-use invite token issued through the admin panel, or
- The deployment's bootstrap token (env `ADMIN_BOOTSTRAP_TOKEN`), used to create the very first admin. The bootstrap path is disabled when that env value is unset. The bootstrap token is itself **single-use**: once it has created one admin, a consumed marker is recorded under the token's hash, so it cannot create a second admin even while the env value remains set.

Tokens are matched by their SHA-256 digest. A stored invite's plaintext is never persisted, and the bootstrap token is never compared in plaintext.

### Headers

```
Authorization: Bearer <invite-token>
Content-Type: application/vnd.api+json
```

### Request body

JSON:API resource of type `AdminUser`:

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `emailAddress` | string | yes | Normalized server-side to trimmed lowercase; must be a syntactically valid email address. |
| `password` | string | yes | 16–256 characters. Stored exactly as sent (not trimmed), so any surrounding whitespace counts toward the length. |

```json
{
    "data": {
        "type": "AdminUser",
        "attributes": {
            "emailAddress": "admin@example.com",
            "password": "a-sufficiently-long-passphrase"
        }
    }
}
```

### Success — `201 Created`

```json
{
    "data": {
        "type": "AdminUser",
        "id": "<user-id>",
        "attributes": {
            "emailAddress": "admin@example.com",
            "userCreationDate": "2026-06-28T12:00:00.000Z"
        }
    }
}
```

- `id` is a server-generated 22-character Base62 identifier.
- `emailAddress` is the normalized (trimmed, lowercased) address.
- `userCreationDate` is an ISO-8601 timestamp assigned at creation.
- The password hash is never returned.

### Errors

| Status | code | Cause |
|---|---|---|
| 415 | `UNSUPPORTED_MEDIA_TYPE_ERROR` | `Content-Type` is not `application/vnd.api+json`. |
| 401 | `UNAUTHENTICATED_ERROR` | Missing or empty `Authorization: Bearer` invite token. |
| 400 | `BAD_REQUEST_ERROR` | Malformed JSON:API envelope or invalid JSON body. |
| 409 | `JsonApiResourceTypeMismatch` | `data.type` is not `AdminUser`. |
| 403 | `InvalidInvite` | Invite token is invalid, expired, revoked, or already used. |
| 422 | `VALIDATION_ERROR` | Email or password missing/invalid (per-field `source`). |
| 409 | `NewUserConflictError` | An admin already exists for that email address. |

Validation `source` values use the form's field names: `email_address` and `password`. Note that `email_address` is snake_case, which differs from the camelCase `emailAddress` request attribute — map errors back to fields accordingly.

### Processing order and behavioral notes for SDK authors

The handler proceeds in this order: media type (415) → bearer token presence (401) → JSON:API envelope (400/409) → invite redeemability (403) → field validation (422) → duplicate-email check and account write (409). Because invite redeemability is checked before field validation, an invalid invite is reported as `InvalidInvite` (403) even when the request body is also invalid.

The invite is **consumed only after** the duplicate-email check passes but **before** the user is written. Consequences:

- A recoverable email conflict (`NewUserConflictError`) does *not* burn the one-time invite token, so the client may retry with a different email using the same token.
- A successful creation consumes the token; one token yields exactly one admin.
- A concurrent second redemption of the same token loses the race and is rejected with `InvalidInvite` (403) rather than creating a duplicate admin.

## 2. Create a Publishing API Token

```
POST /admin-api/v1/publishing-api-tokens
```

The trailing slash is optional.

Mints a Publishing API bearer token on behalf of an existing admin user. The returned plaintext token is shown **once** and is required as the bearer credential for all Publishing API requests.

### Authentication — HTTP Basic admin credentials

```
Authorization: Basic base64("emailAddress:password")
```

- The credentials are decoded as UTF-8; the first `:` separates username from password (passwords may contain `:`).
- The email (username) is canonicalized (trimmed + lowercased) before lookup, mirroring how admin emails are stored, so a non-normalized Basic-auth username still matches.
- A single generic `401 InvalidCredentials` is returned for both an unknown email and a wrong password (no account enumeration). Password-derivation work is performed even for an unknown email so response timing does not reveal whether an account exists.

### Headers

```
Authorization: Basic <base64 credentials>
Content-Type: application/vnd.api+json
```

### Request body

JSON:API resource of type `PublishingApiToken`:

| Attribute | Type | Required | Notes |
|---|---|---|---|
| `permissions` | array | yes | Non-empty array of grant objects. Currently only the allow-all grant is accepted (see below). |
| `timeToLiveSeconds` | integer | no | 1 .. 31,536,000 (365 days). Default 2,592,000 (30 days). |
| `description` | string \| null | no | Operator-facing label. Trimmed; empty → `null`. |

#### Permission grant shape

```json
{ "effect": "allow", "action": ["*"], "resource": "*" }
```

The only `permissions` value accepted at mint time is a **single allow-all grant** exactly as shown above: a one-element array whose sole grant has `effect: "allow"`, `action: ["*"]`, and `resource: "*"`. Any other shape is rejected with `422`.

The permission grammar is designed to grow: future grants will use stable URNs — actions as `urn:kixx:publishing:<capability>:<verb>` and resources as `urn:kixx:publishing:<resource-kind>:<logical-path>` — and the permission evaluator already supports `allow`/`deny` effects, array or string `action`, and wildcard matching. Only the minting-time validation is currently restricted to the allow-all grant.

```json
{
    "data": {
        "type": "PublishingApiToken",
        "attributes": {
            "permissions": [
                { "effect": "allow", "action": ["*"], "resource": "*" }
            ],
            "timeToLiveSeconds": 2592000,
            "description": "CI deploy token"
        }
    }
}
```

### Success — `201 Created`

```json
{
    "data": {
        "type": "PublishingApiToken",
        "id": "<sha256-hex-of-token>",
        "attributes": {
            "token": "kxpat_<64-hex-chars>",
            "permissions": [
                { "effect": "allow", "action": ["*"], "resource": "*" }
            ],
            "description": "CI deploy token",
            "createdBy": "<granting-admin-user-id>",
            "tokenCreationDate": "2026-06-28T12:00:00.000Z",
            "tokenExpirationDate": "2026-07-28T12:00:00.000Z"
        }
    }
}
```

Key facts for SDK authors:

- `attributes.token` is the **plaintext bearer token**, prefixed `kxpat_` followed by 64 lowercase hex characters (256 bits of entropy). It is returned only in this response and never again. Store it securely on receipt.
- The resource `id` is the SHA-256 hex digest of the token. The plaintext token is never persisted.
- `createdBy` records the granting admin's user id for auditing.
- `tokenCreationDate` and `tokenExpirationDate` are ISO-8601 timestamps, where `tokenExpirationDate = tokenCreationDate + timeToLiveSeconds`.

### Errors

| Status | code | Cause |
|---|---|---|
| 415 | `UNSUPPORTED_MEDIA_TYPE_ERROR` | `Content-Type` is not `application/vnd.api+json`. |
| 401 | `UNAUTHENTICATED_ERROR` | Missing or malformed `Authorization: Basic` header. |
| 401 | `InvalidCredentials` | Unknown email or wrong password. |
| 400 | `BAD_REQUEST_ERROR` | Malformed JSON:API envelope or invalid JSON body. |
| 409 | `JsonApiResourceTypeMismatch` | `data.type` is not `PublishingApiToken`. |
| 422 | `VALIDATION_ERROR` | `permissions` not the allow-all grant, TTL non-integer or out of range, or `description` not a string/null (per-field `source`). |

Validation `source` values match the request attribute names: `permissions`, `timeToLiveSeconds`, and `description`.
