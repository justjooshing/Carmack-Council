# Shopify Quality Reference — Carmack × Shopify Platform

Philosophy: John Carmack. Shopify expertise: Shopify Partner Engineering and platform documentation. API version: 2026-01.
Stack context: TanStack Start / TanStack Query / React / TypeScript / Drizzle / Postgres / Redis / Shopify (Admin API, Storefront API, Polaris, Webhooks, Functions).

Every finding must describe the **concrete failure mode** — not just "this is bad practice."
Security patterns are in security.md (Hunt). Backend error handling is in quality-backend.md (Collina). Postgres specifics are in quality-postgres.md (Brandur). This doc covers: Polaris component layer discipline, webhook safety, API surface selection, rate limit handling, embedded app session trust, Shopify Functions, and metafield hygiene.

---

## Principle 1: Polaris — commit to one component layer and never cross the streams

*Carmack: pick one abstraction and be consistent. Two different abstractions doing the same job means two maintenance burdens and undefined behaviour at the boundary.*
*Shopify Platform: "Polaris Web Components are framework-agnostic and the target for new development. Polaris React is fully supported for existing apps."*

Shopify is migrating from Polaris React (`@shopify/polaris`) to Polaris Web Components (`@shopify/polaris`'s web-component surface). The two are not interoperable in the same component tree. Polaris React components render as React-controlled DOM; Web Components register as custom HTML elements. Nesting them causes styling conflicts (CSS custom properties may collide), event bubbling inconsistencies, and focus management breakage.

**Check `conventions.md` before anything else.** If the project specifies a layer, that is the constraint — all new components use it, full stop. If no convention exists, inspect `package.json` for `@shopify/polaris` version and whether existing components use JSX imports or custom element tags.

Web Components use tag syntax: `<ui-page>`, `<ui-card>`, `<ui-button>`. React components use JSX imports: `import { Page, Card, Button } from '@shopify/polaris'`. These look different. Mixing them in the same tree is always a mistake.

### What to check

**Mixing React and Web Component layers**
- JSX Polaris imports alongside `<ui-*>` custom element tags in the same file or component tree.
- Concrete failure: CSS variables applied by one layer are not inherited correctly by the other; focus trapping in modals may break; server-side rendering behaviour differs between the two surfaces.
- Severity: **P1** — produces visual regressions and interaction bugs in production that are hard to reproduce locally.

**Using React components in a project that has migrated to Web Components**
- A new component added with `import { Card } from '@shopify/polaris'` in a repo that has already adopted `<ui-card>`.
- Concrete failure: inconsistent spacing, token overrides, and future migration cost compounds with each new React component added.
- Severity: **P2** — doesn't break today but grows the migration debt with each addition.

**Using Web Components without the Polaris provider setup**
- Web Components require `@shopify/polaris`'s provider stylesheet and the custom elements to be registered before use. Missing setup causes elements to render unstyled or as generic block elements.
- Severity: **P2** — visual breakage; easily caught in development but silently broken in SSR or test environments.

**Reaching outside Polaris for admin UI primitives**
- Using raw `<button>`, `<input>`, `<select>` instead of Polaris equivalents in admin surfaces. Polaris provides keyboard navigation, ARIA roles, dark mode, and merchant locale formatting — all lost with custom elements.
- Severity: **P3** — functional but inconsistent; escalate to **P2** if the custom element lacks accessibility attributes.

---

## Principle 2: Webhooks — HMAC verification is a hard gate, and delivery is not guaranteed

*Carmack: if security checks are optional, someone will skip them.*
*Shopify Platform: "Any webhook handler that does not verify the HMAC signature is accepting events from any sender."*

Shopify signs every webhook with a SHA-256 HMAC using your client secret. The signature is in the `X-Shopify-Hmac-SHA256` header, base64-encoded. Verification requires computing `HMAC-SHA256(clientSecret, rawBody)` and comparing. The comparison must use a timing-safe equality function — string comparison short-circuits on the first mismatch and leaks timing information.

The second problem is delivery reliability. Shopify retries failed webhooks (non-200 response) with exponential backoff for up to 48 hours. Webhook handlers must respond 200 within 5 seconds. Any real work — database writes, API calls, email sends — must be offloaded to a queue. A handler that does inline processing will time out under load, causing Shopify to retry, causing duplicate processing.

The third problem is idempotency. Shopify delivers at-least-once. A `products/update` webhook for the same product may arrive twice, especially under high mutation rate or during retry storms. Without idempotency protection, the handler applies the side effect twice.

### What to check

**Missing HMAC verification**
- A webhook handler that processes the payload without first verifying `X-Shopify-Hmac-SHA256`.
- Concrete failure: any HTTP client can post arbitrary data to the handler. A malicious `orders/paid` event could trigger order fulfilment for non-existent orders. This is an injection attack vector, not a theoretical concern.
- Severity: **P1** — ship nothing without HMAC verification.

**Using string equality instead of timing-safe comparison**
- `receivedHmac === computedHmac` — JavaScript string comparison short-circuits on the first differing character, leaking how many leading bytes match. An attacker who can observe response times can brute-force the HMAC byte-by-byte.
- Fix: use `crypto.timingSafeEqual(Buffer.from(received), Buffer.from(computed))` in Node.js.
- Severity: **P1** on any public-facing webhook endpoint.

**Synchronous processing in the webhook handler**
- Database writes, external API calls, email sends, or any I/O happening inside the handler before the 200 response.
- Concrete failure: at moderate webhook volume, the handler exceeds Shopify's 5-second timeout. Shopify marks the endpoint as failing and begins retries. Retry volume compounds the load. Eventually the app is flagged for webhook removal.
- Fix: validate HMAC, enqueue a job (Redis, Postgres-backed queue), return 200 immediately. Process in a background worker.
- Severity: **P2** for any I/O. **P1** if the handler is already timing out in production.

**Missing idempotency protection**
- A handler that performs side effects (inventory decrement, order record creation, fulfilment trigger) without checking whether this webhook has already been processed.
- Fix: persist `{topic, shopify_event_id}` (use the `X-Shopify-Webhook-Id` header) to a `processed_webhooks` table on first delivery. On subsequent delivery, check existence and return 200 without processing.
- Severity: **P2** — silent data corruption under retry conditions. **P1** for financial operations (orders, payments).

---

## Principle 3: API surface selection — each API has a lane, and crossing lanes has consequences

*Carmack: use the interface that's designed for your access pattern. Using the wrong one is technical debt that manifests as security vulnerabilities and rate-limit headaches.*

Three APIs, three distinct use cases:

- **Admin API (GraphQL)** — full merchant data access. Requires an access token with specific scopes. Server-side only. Never expose the token to the client.
- **Storefront API (GraphQL)** — read-optimised product, collection, cart, and checkout data. Uses a public storefront access token. Client-safe, but the token is not a secret — assume it's visible. Scope it to the minimum required permissions.
- **Ajax API** — theme-only cart operations via `/cart/add.js`, `/cart/change.js`. No authentication required. Works within a Shopify storefront session.

The Admin REST API is in active deprecation. New code should always use the Admin GraphQL API.

### What to check

**Admin API token exposed client-side**
- The Admin API access token (`X-Shopify-Access-Token`) appearing in client-side JavaScript, environment variables accessible to the browser, or API routes that proxy the token without authentication.
- Concrete failure: any user who opens DevTools can extract the token. With full `write_products` scope, they can delete all products. The token is long-lived.
- Severity: **P1** — immediate revoke and rotate if found in production.

**Using REST Admin API instead of GraphQL**
- New code making requests to `/admin/api/2026-01/products.json` instead of the GraphQL endpoint.
- Concrete failure: REST endpoints are being deprecated. Some REST endpoints are already removed. REST responses over-fetch (you get all product fields when you need title and price), consuming response payload budget and increasing parse time.
- Severity: **P2** for new code. **P3** for existing code with no immediate plans to migrate.

**Storefront API used server-side for data that requires Admin access**
- Attempting to fetch draft orders, customer PII, or inventory levels via the Storefront API because it's "easier to set up."
- Concrete failure: the Storefront API doesn't expose this data — calls silently return empty results or null fields, producing hard-to-diagnose data gaps.
- Severity: **P2** — wrong API for the job; will fail silently rather than loudly.

**Storefront access token over-scoped**
- Creating a Storefront API token with all permissions enabled because it's convenient.
- Concrete failure: the token is public by design. Over-scoped tokens expose customer account operations, checkout mutations, or B2B data to any client who knows the store URL.
- Severity: **P2** — scope tokens to the minimum required for the feature.

**Unpinned API version**
- Code using `api_version` set to `unstable` or always using the latest — `https://{store}.myshopify.com/admin/api/unstable/graphql.json`.
- Concrete failure: Shopify makes breaking changes to the unstable version without notice. The stable API version is supported for 12 months. Unpinned code breaks on Shopify's release schedule, not yours.
- Severity: **P2** for `unstable`. **P3** for not tracking which stable version is in use and when it will be sunset.

---

## Principle 4: Rate limiting — the quota model is a hard constraint and requests will be dropped

*Carmack: if you can saturate a rate limit, you will, under production load.*
*Shopify Platform: "After a query returns a throttled error, wait and retry. Shopify does not queue your requests."*

The Admin GraphQL API uses a calculated cost model: each query has a cost based on the number of fields and connections requested. The default bucket size is 1,000 points; it restores at 50 points per second. Deeply nested queries (products → variants → metafields) cost more. Shopify includes `extensions.cost` in every response — `actualQueryCost`, `throttleStatus.currentlyAvailable`, `throttleStatus.restoreRate`. Ignoring this is flying blind.

The Storefront API uses request-based rate limiting: 60 seconds of compute time per minute per storefront, distributed across requests.

A 429 from Shopify is a hard stop — the request was not queued, not partially processed. It must be retried with backoff.

### What to check

**No handling for 429 responses**
- API calls that surface a generic error or throw on 429 without retrying.
- Concrete failure: under production load (bulk import, automated sync), the app fails silently. Merchants see partial syncs, missing products, incomplete order processing.
- Fix: exponential backoff with jitter starting at 1 second. Respect `Retry-After` header if present.
- Severity: **P1** for any code path that calls the Admin API without 429 handling.

**Not reading `extensions.cost` in Admin GraphQL responses**
- Code that fires and forgets, never monitoring how close it's pushing to the rate limit.
- Concrete failure: a background sync job slowly saturates the bucket without triggering 429s until peak load, when simultaneous user-facing requests tip it over.
- Fix: log `throttleStatus.currentlyAvailable` per response. Add a delay when `currentlyAvailable` drops below 200 points.
- Severity: **P2** for background jobs. **P3** for low-frequency user-triggered queries.

**Querying more fields than needed**
- `query { products(first: 250) { edges { node { ... all fields ... } } } }` when only `title` and `handle` are needed.
- Concrete failure: over-fetched queries consume disproportionate query cost per call, reducing how many calls can be made per second. A 250-product query with variants and metafields can cost 500+ points.
- Severity: **P3** — select only the fields you need in every GraphQL query.

**Bulk operations not used for large data sets**
- Paginating through thousands of products/orders/customers using repeated GraphQL queries rather than Shopify's Bulk Operations API.
- Concrete failure: 10,000 products at 250 per page = 40 API calls. Each call costs quota. Bulk Operations run server-side at Shopify's infrastructure cost, not yours, and return a JSONL file via a temporary URL — zero quota impact.
- Severity: **P2** for any sync that touches more than ~1,000 records.

---

## Principle 5: Embedded apps — App Bridge is the trust boundary, not cookies

*Carmack: know exactly where your trust boundaries are. Every assumption about the calling context that you can't verify is a vulnerability.*
*Shopify Platform: "Session tokens replace the previous cookie-based authentication. Third-party cookies are blocked by most browsers in embedded contexts."*

Shopify apps embedded in the merchant admin run inside an `<iframe>`. Third-party cookies are blocked by default in Safari and Chrome (in most configurations). Any app that still relies on cookie-based sessions in the embedded context will silently fail for a significant portion of merchants.

App Bridge v4+ provides session tokens: short-lived (1-minute TTL) JWTs signed by Shopify. The token is fetched client-side via App Bridge and sent as a bearer token to the app's backend. The backend verifies the token using the app's client secret. The token contains the `dest` claim (shop domain), `sub` claim (user ID), and `exp` claim (expiry) — all must be validated.

### What to check

**Cookie-based session authentication in embedded context**
- Code that sets `Set-Cookie` on the top-level domain and reads it in the embedded iframe.
- Concrete failure: Safari blocks third-party cookies by default. Affected merchants see blank screens or redirect loops. The failure is browser-version-dependent and hard to reproduce in Chrome without privacy settings.
- Fix: migrate to App Bridge session tokens. Use Shopify's `authenticate.admin()` helper (if using the Shopify Remix template) or manually verify the session token JWT on each request.
- Severity: **P1** for any app still using cookie-based auth in the embedded admin.

**Session token not verified server-side**
- The server accepts the App Bridge session token without verifying the JWT signature and claims.
- Concrete failure: any client can forge a session token by constructing a JWT without a valid signature. Without server-side verification, the forged token grants access to any merchant's data.
- Verify: decode the JWT header to get the algorithm (always HS256 for Shopify session tokens), verify the signature with `HMAC-SHA256(clientSecret, header.payload)`, validate `exp` (not expired), `iss` (matches `https://{shop}.myshopify.com/admin`), and `dest` (same shop).
- Severity: **P1** — this is an authentication bypass.

**Using localStorage or sessionStorage for sensitive app state in embedded context**
- Storing access tokens, session identifiers, or merchant data in `localStorage` inside the embedded app.
- Concrete failure: `localStorage` is accessible to any script on the same origin. In an embedded iframe, the origin is the app's own domain — but browser extensions and XSS can still read it.
- Severity: **P2** — store sensitive state server-side, not in browser storage.

---

## Principle 6: Shopify Functions — hard limits are enforced at runtime, not at deploy

*Carmack: when execution time is capped and non-negotiable, architecture decisions become correctness decisions.*
*Shopify Platform: "Functions that exceed the 5ms CPU time limit are terminated without completing."*

Shopify Functions replace Shopify Scripts (sunset August 2025). Functions run in a sandboxed WebAssembly environment with hard constraints: 5ms CPU time, no network access, no filesystem access, deterministic execution only. The Function receives an `input` object from Shopify's GraphQL schema and returns an `output` object. All data needed for computation must be present in the input — there is no way to fetch additional data mid-execution.

The correct architecture: design the Function's input query to fetch everything it might need upfront. Perform all computation against the input only. Return the output. Any logic that requires external data must be moved to a Shopify metafield (written by your app, readable in the Function's input query) or a custom attribute on the cart.

### What to check

**Shopify Scripts still in use**
- Any reference to `shopify/scripts`, Script Editor, or script-based discount/shipping/payment customisations.
- Concrete failure: Scripts were sunset in August 2025. Existing scripts may have been automatically disabled. New script deployment is not possible.
- Severity: **P1** — migrate to Shopify Functions immediately.

**Functions with unbounded computation**
- Loops over the full `lines` array without a guard, nested iterations over product tags and metafield values, or string operations proportional to cart size.
- Concrete failure: a cart with 100+ line items and complex metafields can push computation over the 5ms limit. The Function terminates mid-execution; Shopify applies no discount/shipping rule rather than a partial one — silently producing the wrong merchant outcome.
- Severity: **P2** — profile with representative cart sizes. Add early-exit guards.

**Attempting network access from within a Function**
- Code that tries to call `fetch()`, use the Admin API, or read from Redis inside a Function.
- Concrete failure: Functions run in a sandboxed WASM environment with no network access. The code compiles but throws at runtime. Data that a Function needs must be pre-loaded via metafields or cart attributes.
- Severity: **P1** — won't work; redesign required.

---

## Principle 7: Metafields and Metaobjects — register types, use namespaces correctly, avoid JSON blobs

*Carmack: untyped storage is unstructured state. Unstructured state is a liability that compounds.*

Metafields are key-value storage attached to Shopify resources (products, variants, customers, orders). They can be registered with type definitions — registered metafields have schema validation, are accessible in Liquid natively, and can be exposed in Storefront API responses. Unregistered metafields are wild-card strings that bypass all validation.

Metaobjects are structured objects with defined schemas — use them instead of cramming JSON into a `json` metafield when the data has multiple fields with relationships.

Namespace conventions matter. Shopify reserves `$app:` namespace prefix for app-owned metafields that merchants cannot directly edit. The `custom` namespace is for merchant-editable fields. Using `custom` for app-critical fields means merchants can corrupt them.

### What to check

**Unregistered metafields used for structured data**
- Writing to ad-hoc `namespace.key` combinations without a metafield definition registered via the Admin API.
- Concrete failure: no type validation — any string can be written to a `single_line_text` metafield that expects an ISO date. Storefront API responses may not include unregistered metafields. Liquid access requires `metafields` permission on the theme.
- Severity: **P2** — always register metafield definitions before writing to them in production.

**JSON metafields for data with a known schema**
- Storing `{ "feature_flags": {"show_badge": true, "tier": "gold"} }` as a `json` metafield instead of defining a Metaobject with typed fields.
- Concrete failure: no validation, no queryable sub-fields via GraphQL, no merchant-friendly UI in the Shopify admin. Parsing errors on malformed JSON surface as runtime exceptions in app code, not at write time.
- Use `json` metafields for: truly opaque blobs, webhook payloads, external system snapshots. Use Metaobjects for: structured data with defined fields, reusable content types, data that merchants will edit.
- Severity: **P2** for structured data stored as JSON.

**Using `custom` namespace for app-owned fields**
- Writing app-critical configuration or state to the `custom` namespace.
- Concrete failure: merchants can edit or delete `custom` metafields directly in the Shopify admin. An app that depends on `custom.loyalty_tier` for access control will behave incorrectly if a merchant sets it to an invalid value or deletes it.
- Fix: use the `$app:` reserved namespace for fields the app owns. `$app:loyalty_tier` cannot be modified by merchants.
- Severity: **P2** for app-owned fields in the `custom` namespace. **P1** if the field affects security or access control.

**Reading metafields without checking for null**
- Code that reads `product.metafield(namespace: "app", key: "config")` and immediately accesses `.value` without null-checking.
- Concrete failure: not all resources have all metafields set. A new product that hasn't been configured by the app will return `null` for the metafield. `null.value` throws and crashes the request handler.
- Severity: **P2** — always null-check metafield access.

---

## Gaps: What This Doc Doesn't Cover

- **Webhook security (HMAC, injection)**: HMAC verification covered in Principle 2 above. General injection patterns are in **security.md (Hunt)**.
- **Liquid templating**: theme development with Liquid is a separate concern from app development. This doc focuses on the app and API layer.
- **Hydrogen**: Shopify's headless storefront framework (built on React Router 7). Performance and data fetching patterns for Hydrogen are covered by TanStack Query and the Performance expert.
- **Checkout Extensibility**: checkout UI extensions are a deep topic with their own constraints. Treat UI extension development as a separate review domain.
- **Billing API**: Shopify app billing (recurring charges, usage charges) has its own patterns. Not covered here.
- **App Store listing requirements**: compliance review, not code review.

---

## Quick Reference: Severity Guide

| Severity | Pattern | Examples |
|----------|---------|----------|
| **P1 — Fix Now** | Data loss, auth bypass, guaranteed production failures | Admin API token exposed client-side, missing HMAC verification on webhooks, cookie auth in embedded context (Safari breaks), session token not verified server-side, Shopify Scripts still deployed, Function attempting network access, string equality for HMAC comparison |
| **P2 — Fix Soon** | Silent failures, data corruption under load, quota exhaustion | Mixing Polaris React and Web Components, synchronous webhook processing, missing idempotency on webhooks (financial), no 429 handling on Admin API calls, unregistered metafields for structured data, JSON metafields for typed data, app fields in `custom` namespace, null-unchecked metafield reads |
| **P3 — Consider** | Developer experience, future-proofing, hygiene | REST Admin API for new code, unpinned API version, over-scoped Storefront token, unbounded GraphQL field selection, missing `extensions.cost` monitoring, Polaris React in a Web Component project |

### The Overriding Filter

Before writing any finding, apply the Carmack–Shopify synthesis:

1. **Is the webhook HMAC verified before any action is taken?** If not, it's P1. No exceptions. An unverified webhook handler is an open injection endpoint.
2. **Is the Admin API token contained server-side?** If it appears in client code, environment variables accessible to the browser, or public repositories, it's P1 and must be rotated.
3. **Is this the right API for this access pattern?** Admin data server-side = Admin API. Public-facing product/cart data = Storefront API. Wrong API = either a security issue or a silent data gap.
4. **Does this Shopify Function assume it can do I/O?** If yes, it's P1 — the architecture is wrong. All data must be in the input query.
5. **Is the Polaris component layer consistent with `conventions.md`?** If mixed, flag as P1. Two component layers in the same tree produce unpredictable behaviour at the boundaries.
