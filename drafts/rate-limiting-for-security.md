## Rate Limiting & Lockout

Sensitive unauthenticated routes (admin login, invite-only signup, invite-token
lookups) are throttled with temporary, self-resetting lockouts. Like CSRF, this
is a presentation-layer security control: request handlers apply it, and
Transaction Scripts stay unaware of it.

**Where policy lives.** Thresholds are configuration, not code. Each surface has
a policy block under `RATE_LIMIT` in `node-config.json`, read at request time
through `context.config.env.RATE_LIMIT` (the same `config.env` bundle Hyperview
reads). A policy is `{ maxFailures, windowSeconds, cooldownSeconds }`. The
`app/presentation/lib/rate-limit.js` helpers read these blocks (falling back to
in-code defaults if a block is missing) so handlers never read config or build
storage keys directly.

**Mechanics.** Counters live in the eventually-consistent Key/Value Store via the
`RateLimit` collection, keyed by an opaque scope id. The window is *sliding*:
each failure refreshes the record TTL to `windowSeconds`, so a scope that stops
failing resets on its own. Once `failureCount` reaches `maxFailures`, the scope
locks for `cooldownSeconds` with a matching TTL, so the lock clears itself with
no follow-up write. Because the Key/Value Store has no concurrency control, the
read-modify-write increment can lose updates and slightly undercount under a
concurrent attack — acceptable fail-soft behavior for throttling, and the reason
this does not use the document store's optimistic-concurrency path.

**Login keying.** Login is limited on two scopes at once: per-IP (catches one
source spraying many accounts) and per-`(IP, email)` (catches a focused attack on
one account). Keying the account scope on the IP as well means an attacker can
never lock a real admin out from the admin's own IP — there is deliberately no
standalone per-email scope, which would otherwise be a denial-of-service lever.

**Handler pattern.** Check before doing protected work; record after detecting a
failure; clear on success. Re-render the form (or the page's no-form state) with
a non-enumerating "try again later" message rather than returning a hard `429`:

```js
import {
    checkLoginThrottle,
    recordLoginFailure,
    clearLoginThrottle,
    throttleMessage,
} from '../lib/rate-limit.js';

const throttle = await checkLoginThrottle(context, request, form.email_address);
if (throttle.throttled) {
    return response.updateProps({
        form: await getCsrfFormContext(context, request, response, form),
        throttled: true,
        throttleMessage: throttleMessage(throttle.retryAfterSeconds),
    });
}
// ... on InvalidCredentials: await recordLoginFailure(...)
// ... on success: await clearLoginThrottle(...)
```

Only count genuine abuse signals: the invite-GET limiter records a failure only
when a presented token matched no stored invite, so legitimate visitors and
expired-link clicks (which still resolve to a real record) are never penalized.

Templates render the message from a `{{#if throttled}}` callout:

```html
{{#if throttled}}
<div class="callout callout--error" role="alert">
    <div class="callout__body"><p class="type-body-sm">{{ throttleMessage }}</p></div>
</div>
{{/if}}
```

To throttle another HTML route, add a `RATE_LIMIT` policy block, add scope-key
and helper functions to `rate-limit.js`, and call them from the handler — do not
re-implement the counter logic.

