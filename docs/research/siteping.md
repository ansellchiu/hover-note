# SitePing: server contract and anchor format

Research ticket: "What do SitePing's server contract and anchor format look like?"

- Source read: `github.com/NeosiaNexus/SitePing` at commit `fd64185` (2026-11-16), full history fetched. Paths below are relative to that repo; `L` = line numbers at that commit.
- Versions at that commit: `@siteping/widget` 0.12.5, `@siteping/server` 0.2.4, `@siteping/adapter-prisma` 0.7.1, `@siteping/adapter-drizzle` 0.2.4, `@siteping/dashboard` 0.3.2.
- Method: static reading only. Nothing was installed, built or run. Statements marked **Inference** are conclusions drawn from the code, not behaviour I observed.

## Short answer

Yes, a different widget can use SitePing's server and stores, but **not with an opaque Anchor payload**. The server validates a fixed anchor shape with zod, the stores persist it as typed columns, and there is no free-form metadata field anywhere on a feedback or annotation. An adapter has two honest options: translate hover-note's Anchor into SitePing's nine anchor fields plus rect and viewport numbers (lossy for text ranges), or send zero annotations and lose the anchor entirely. Everything else (threads, statuses, auth, webhooks) maps well. The dashboard tolerates both options.

## 1. HTTP API of `@siteping/server`

One endpoint, five methods, built on Fetch `Request`/`Response`. `createSitepingHandler(options)` returns `{ GET, POST, PATCH, DELETE, OPTIONS }` (`packages/server/src/handler.ts` L173-210). An OpenAPI document ships as `@siteping/server/openapi.json` (path `/api/siteping`, protocol version 1).

Comments share the feedback endpoint. Routing is by body key: a POST body with `feedbackId` is a comment, a DELETE body with `commentId` is a comment delete (`handler.ts` L53-67).

| Method | Body / query | Success | Source |
|---|---|---|---|
| `GET ?projectName=` | filters `page`, `limit` (max 100, default 50), `type`, `status`, `statuses` (CSV), `search` (matches `message`), `url`, `urlPattern` | `200 { feedbacks, total, protocol: 1, capabilities: { comments, deleteComments }, permissions: { canDeleteAll } }` | `validation.ts` L206-226; `operations/list-feedbacks.ts` L55-76 |
| `POST` (feedback) | `FeedbackPayload`, see below | `201` feedback; replay of the same `clientId` returns the stored record; `409` if that `clientId` belongs to another project | `validation.ts` L128-170; `operations/create-feedback.ts` L33-51, L162-164 |
| `POST` (comment) | `{ projectName, feedbackId, body, authorName, authorEmail, authorRole, clientId }` | `201` comment; `501` if the store has no `addComment`; `409` past 100 client comments | `validation.ts` L174-187; `operations/create-comment.ts` L41-64 |
| `PATCH` | `{ id, projectName, status }` only | `200` feedback | `validation.ts` L195-199; `operations/update-feedback.ts` L32-58 |
| `DELETE` | `{ id, projectName }`, or `{ projectName, deleteAll: true }`, or `{ projectName, feedbackId, commentId }` | | `validation.ts` L189-204 |
| `OPTIONS` | | `204` preflight | `cors.ts` L67-78 |

Things the API does not have (observed by absence in `handler.ts` and `validation.ts`): get-one-by-id, edit a feedback's message, edit a comment, edit or replace an anchor, an "updated since" filter, push or streaming. PATCH changes status and nothing else.

Errors: `400 { errors: [{ field, message }] }` for validation (max 20 issues), otherwise `{ error }` with 401, 403, 404, 409, 413, 415, 422, 500, 501, 502 (`constants.ts` L26-47, `pipeline.ts` L186-199).

Every response strips `clientId`, and blanks `authorEmail` unless the caller may read it (`pipeline.ts` L63-91).

### Auth

Two mutually exclusive policies (`options.ts` L247-332).

**`apiKey` (shared secret)**, `access.ts` L85-153:
- Sent as `Authorization: Bearer <key>`, compared in constant time.
- With a key set, `publicEndpoints` defaults to `["POST", "OPTIONS"]`. So GET, PATCH and DELETE need the key; POST (create feedback, create comment) is public.
- With no key: GET and POST are open; PATCH and DELETE answer 401 unless `requireAuthForDestructive: false`. In `NODE_ENV=production` with no key the factory throws at startup.
- Emails are redacted for requests without the key (`redactUnauthenticatedEmails`, default true).

**`access` (custom)**, `options.ts` L103-108: `authenticate(request)` returns a principal or 401; optional `authorize({ action, projectName, feedbackId, commentId, dryRun })` returns false for 403; `canReadAuthorEmail`; `canCommentAsTeam`. Actions are `create | list | update | delete | deleteAll | createComment | deleteComment`. `authorize` requires a store with `verifyProjectOwnership` (`handler.ts` L137-143). Mutations get CSRF guards: body must be `application/json` (415), and with `allowedOrigins` a foreign `Origin` is refused (403); requests with no `Origin` pass (`csrf.ts` L26-43).

Every feedback in a response carries `permissions: { canChangeStatus, canDelete, canComment, canDeleteComment }`, computed by dry-running the policy (`pipeline.ts` L241-250).

### CORS

Off by default. With no `allowedOrigins`, no CORS headers are sent and browsers block cross-origin calls. With a list, exact-match origins are reflected with `Allow-Credentials: true`, methods `GET, POST, PATCH, DELETE, OPTIONS`, headers `Content-Type, Authorization` plus whatever the preflight asks for (`cors.ts` L14-31, L50-60). No wildcard option.

### Hooks

- Rewrite before store: `beforeCreate(input, ctx)`, `beforeComment(input, ctx)`; both run before `authorize`.
- Rewrite before response: `presentFeedback(record, ctx)`.
- Lifecycle: `onCreated`, `onUpdated`, `onDeleting` (throw to abort, answers 502), `onDeleted`, `onCommentCreated`. Awaited before the response; a throw is logged, never fails the request (`options.ts` L128-151, L196-225).
- Rate limiting is explicitly not provided (`handler.ts` L74-77).

### Webhooks

`webhooks: WebhookConfig | WebhookConfig[]` with `{ url, type: "slack" | "discord" | "generic", headers, timeoutMs (default 5000), events, onError }` (`webhooks.ts` L95-102). Events: `feedback.created` (default), `feedback.status_changed`, `feedback.deleted`, `comment.created`, named in an `X-Siteping-Event` header (L52-67). Generic body is the stored record minus `clientId`, with the author email unredacted. Delivery is fire-and-forget, one attempt, no retry and **no built-in signature**; `headers` is the only hook for a signing scheme. `feedback.status_changed` fires on every PATCH, including a no-op one.

## 2. Data model

`SitepingStore` is the store interface (`packages/core/src/types.ts` L1124-1200): `createFeedback`, `getFeedbacks`, `findByClientId`, `updateFeedback`, `deleteFeedback`, `deleteAllFeedbacks`, `verifyProjectOwnership`, and optional `createFeedbackIfAbsent`, `addComment`, `addCommentIfAbsent`, `deleteComment`.

Three models, defined once in `packages/core/src/schema.ts` L115-213 and compile-time locked to the record types (L232-247):

- **`SitepingFeedback`**: `id`, `projectName`, `type`, `message` (max 5000), `status`, `url`, `urlPattern?`, `screenshotUrl?`, `screenshotRegion?` (JSON), `diagnostics?` (JSON), `viewport`, `userAgent`, `authorName`, `authorEmail`, `clientId` (unique), `resolvedAt?`, `createdAt`, `updatedAt`.
- **`SitepingAnnotation`** (many per feedback, cascade delete): the anchor and rect as flat typed columns, see section 3.
- **`SitepingComment`** (many per feedback, cascade delete): `id`, `feedbackId`, `body` (max 5000), `authorName`, `authorEmail` (may be `""`), `authorRole`, `clientId` (unique), `createdAt`.

Mapping to hover-note vocabulary (**Inference**): a SitePing "feedback" is an Annotation whose first Comment is the `message` field; later Comments are `comments[]`. So the Thread is split across two shapes, and the first Comment cannot be deleted or edited separately from the Annotation.

**Stores**: Prisma (`adapter-prisma`, peer `@prisma/client` 5-7), Drizzle Postgres and libSQL (`adapter-drizzle`), in-memory (`adapter-memory`, zero deps), `localStorage` (`adapter-localstorage`, browser-only). `adapter-kit` provides a conformance test suite for third-party stores. Memory and localStorage share one engine in core (`createCollectionStore`).

**Types and statuses** (`types.ts` L578-598): `type` is required, one of `question | change | bug | other`. `status` is `open | in_progress | resolved | wont_fix`; the last two are "closed" and stamp `resolvedAt`. New feedback is always stored `open` (`create-feedback.ts` L38). Any status can move to any other; there is no transition table.

**Who can change status**: whoever passes the gate for PATCH. Under `apiKey` that is the key holder only (by default). Under `access` it is whatever `authorize({ action: "update" })` allows. The store keeps no record of who changed it.

**Comment roles** (`types.ts` L847-848): `client` or `team`. `team` is a claim the server keeps only for the key holder or when `canCommentAsTeam` says so; otherwise it is silently downgraded to `client` (`create-comment.ts` L51-52). There is no "agent" role and no author id, only name and email.

**Screenshots**: the widget sends `screenshotDataUrl` (JPEG, PNG or WebP base64 data URL, max 1.5 MB) and optional `screenshotRegion` (`{xPct,yPct,wPct,hPct}`, strict object) (`validation.ts` L155-165). The store either uploads through a `ScreenshotStorage` (`upload(dataUrl, { feedbackId, mimeType }) -> { url }`, optional `delete`) or stores the data URL inline in `screenshotUrl` with a warning (`core/src/screenshot-storage.ts` L36-76; `adapter-prisma/src/index.ts` L373-412, L500-505). `@siteping/screenshot-storage` ships S3-compatible, Cloudflare Images, DB-table, filesystem and memory backends. A failed upload saves the feedback without a screenshot.

## 3. What the widget stores as its anchor

Per annotation (`types.ts` L1291-1365, flattened into columns by `flattenAnnotation` L1042-1064):

| Field | Meaning |
|---|---|
| `cssSelector` | Selector from `@medv/finder`, tuned to skip CSS-in-JS class hashes and framework ids; prefers `data-testid`, `data-id`, `role`, `aria-label`. Shadow DOM: one selector per tree joined by `" >>> "`. |
| `xpath` | Positional path, max 6 levels, shortcut on ids. |
| `elementTag`, `elementId?` | Tag name; id if present and at most 191 chars. |
| `textSnippet` | First 120 chars of the element's text. |
| `textPrefix`, `textSuffix` | About 32 chars from the nearest sibling with text, before and after. |
| `neighborText` | 40 chars each of previous and next sibling. |
| `fingerprint` | `"childCount:sameTagSiblingIndex:attrHash"`, hash over `role, aria-label, type, name, href, src, data-testid, data-id`. |
| `anchorKey?` | Value of the closest ancestor's `data-feedback-anchor` attribute, a host opt-in stable key. |
| `xPct, yPct, wPct, hPct` | The drawn rectangle as fractions of the anchor element's box. |
| `scrollX, scrollY, viewportW, viewportH, devicePixelRatio` | Capture-time context. |

Feedback-level context: `url` (default `location.pathname`), `urlPattern`, `viewport` (`"1280x720"` string), `userAgent`.

Sources: `packages/widget/src/dom/anchor.ts` L18-27, L62-112; `dom/fingerprint.ts`; `dom/xpath.ts`; `dom/text-context.ts`.

The anchor is always **one element plus a rectangle inside it**. There is no text-range anchor: no start or end offsets, no quote-with-offsets selector. For a drawn rectangle, the anchor element is the smallest ancestor of the element at the rectangle's centre that contains the whole rectangle (`anchor.ts` L134-155).

### Re-finding the element

`resolveAnchor` in `packages/widget/src/dom/resolver.ts` L215-279. This logic lives **only in the widget**; server and dashboard never resolve anchors.

1. Gather candidates from every strategy: `[data-feedback-anchor="key"]`, `[id="..."]`, the CSS selector, the XPath. Up to 16 per strategy; tag must match except for `anchorKey` (L343-408).
2. Score each candidate: `strategy prior x verification x visibility`. Priors: anchorKey 1.0, id 1.0, css 0.95, xpath 0.9, scan 0.85 (L62-68). Verification blends text snippet, fingerprint, prefix/suffix and neighbour text, with whitespace-normalised fuzzy matching. Hidden elements are penalised, not excluded.
3. If nothing beats 0.85, sweep same-tag elements (capped at 10,000, top 24 fully scored), under a per-pass budget.
4. Accept or orphan: selector hits need text score at least 0.5 or a strong structural signal; scan hits need blend at least 0.4 (L98-100, L528-540). No acceptable candidate means the annotation is orphaned (`null`).
5. Confidence is returned with the element; the marker UI flags below 0.7 as approximate (comment at L106-109).

The rect is then re-projected from the percentages onto the element's current box (L664-687).

## 4. How tightly server and dashboard depend on the anchor shape

**Server: tightly.** `annotationSchema` (`validation.ts` L43-82):

- Required, non-empty strings: `cssSelector` (1-2000), `xpath` (1-2000), `elementTag` (1-191).
- Required strings that may be empty: `textSnippet` (max 500), `textPrefix`, `textSuffix`, `fingerprint` (max 200), `neighborText` (max 500).
- Optional: `elementId`, `anchorKey` (max 200).
- Required numbers: `rect.{xPct,yPct,wPct,hPct}` each in [0, 1]; `scrollX`, `scrollY`; `viewportW`, `viewportH` positive integers; `devicePixelRatio` positive (default 1).
- The schema is `z.object`, not `z.strictObject` (compare `screenshotRegionSchema`, L110). **Inference**: in zod 4 that strips unknown keys, so extra anchor fields are silently dropped, neither rejected nor stored.
- The schema is compile-time locked to `FeedbackPayload` (L281-285), so the maintainers treat this shape as the wire contract.

The server checks **shape only**. It never parses the selector or XPath, never checks they are valid CSS or XPath. They are opaque strings to it.

**`annotations` may be empty.** The array is bounded at 50 with no minimum (`validation.ts` L149). A feedback with zero annotations is a valid POST.

Other required feedback fields an adapter must supply: `type` (enum), `message` (1-5000), `url` (1-2000), `viewport` (1-50), `userAgent` (1-500), `authorName` (1-200), `authorEmail` (must be a valid email on a feedback; `""` is allowed only on comments), `clientId` (`^[a-zA-Z0-9_-]+$`, max 191, globally unique).

**Stores: tightly.** Anchor fields are typed columns, not a JSON blob (`schema.ts` L152-188). The only JSON columns are `screenshotRegion` and `diagnostics`, both with fixed shapes validated upstream.

**Dashboard: loosely.** It reads `annotations[0]` for `devicePixelRatio` in a caption, and when there is no screenshot shows `cssSelector` (click to copy) and `textSnippet`; extra annotations get the same card. With zero annotations and no screenshot it shows a "no screenshot" placeholder (`packages/dashboard/src/components/evidence-card.tsx` L49-100, L124-126, L196-211). It does not resolve anchors or render a live page.

**Issue-tracker integration** (`integration-issues`) prints `elementTag`, `cssSelector` and `textSnippet` into GitHub/GitLab issue bodies and skips the section when there are no annotations (`core/issue-format.ts` L82-92).

## 5. Can a different widget use the server with its own Anchor?

**With a truly opaque payload: no.** There is nowhere to put it. Unknown keys are dropped, and no model has a metadata column.

**With translation: yes.** What the adapter must do:

| hover-note | SitePing | Notes |
|---|---|---|
| Annotation | feedback | Must invent `type` (e.g. always `other`), `viewport`, `userAgent`. |
| Annotation id | `id` (server-generated) plus `clientId` (ours, idempotency key) | `clientId` is never returned in responses, so the adapter must keep its own id-to-`id` mapping from the POST response. |
| First Comment | `message` | Cannot be edited or deleted on its own. |
| Later Comments | `comments[]` via POST with `feedbackId` | No edit. Delete needs permission. Max 100 `client` comments per thread. |
| Comment author (human or agent) | `authorName`, `authorEmail`, `authorRole` | Feedback needs a syntactically valid email. No agent role; `team` needs the key or a policy. |
| Thread lifecycle | `status` | Four fixed values. |
| Anchor | one `annotations[]` entry | Fill `cssSelector`, `xpath`, `elementTag` with real or placeholder non-empty values; rect `{0,0,1,1}` and real viewport numbers. |
| Page | `url` | Any non-empty string up to 2000 chars; used as an exact-match filter. |

Three ways to carry hover-note's Anchor, in order of how much I would trust them:

1. **Native mapping.** Put hover-note's selector into `cssSelector`, its text quote into `textSnippet`/`textPrefix`/`textSuffix`. The dashboard and issue integration then show something meaningful. Loses anything that does not fit (text-range offsets, alternative selectors).
2. **Smuggle the serialised Anchor in `xpath`** (opaque to the server, 2000 chars, a `Text` column, shown nowhere in the dashboard). **Inference**, not tested. It works only as long as nothing evaluates the field: SitePing's own widget would run it through `document.evaluate` and catch the error (`resolver.ts` L396-405), so mixed deployments degrade rather than crash. It is a hack against a field whose meaning the maintainers may tighten.
3. **Skip the HTTP handler** and implement our own `SitepingStore`-backed endpoint, or call a store directly server-side. Still no opaque column in the Prisma/Drizzle schemas, so this only helps if we also own the schema, at which point we are not really using SitePing's store.

What breaks or degrades:

- **Text-range anchors**: no representation. Must be reduced to the containing element plus a rect, or smuggled.
- **Dashboard evidence card**: works. Shows our selector and snippet, or the screenshot if we send one. Garbage in `cssSelector` shows as garbage.
- **Screenshots**: optional. If hover-note sends none, the dashboard falls back to the anchor view. If it sends one, it must be a base64 data URL up to 1.5 MB; `screenshotRegion` is optional. hover-note's 10 KB core cannot capture screenshots itself (SitePing lazy-loads `html2canvas-pro`, about 60 KB gzip by its own doc comment, `types.ts` L262).
- **SitePing's widget on the same project**: would try to resolve our annotations with its resolver. With native mapping it may even pin them; with placeholders they orphan.
- **Sync**: polling `GET` only, 100 per page, no change feed. Webhooks are server-to-server, single attempt.
- **Git-file default vs SitePing**: SitePing owns ids and timestamps; round-tripping between Backends needs an id map.

## 6. Widget bundle size, dependencies, touch mode

**Size** (budgets enforced by `size-limit`, gzipped, each "measured size plus about 1 kB"; `packages/widget/.size-limit.mjs`):

- ESM first load: 52 kB budget (README says about 51 kB).
- Panel, lazy on first open: 22 kB.
- Eight non-English locales, lazy: 17 kB total.
- Single-file IIFE for `<script>` use: 146 kB.
- Screenshot engine `html2canvas-pro`: separate lazy chunk, about 60 kB gzip per the config doc comment.

I did not build or measure; these are the repo's own budgets.

**Dependencies**: `@medv/finder` (bundled in) and `html2canvas-pro` (dynamic import); optional peer `react` 18/19 for the `/react` entry. `@siteping/server` depends only on `zod` 4 and needs Node 22+. `@siteping/core` is private and bundled into each package.

So SitePing's first load is roughly five times hover-note's whole budget, which supports replacing the widget rather than wrapping it.

**Touch mode** (`packages/widget/src/annotator.ts`):

- Touch is detected with `matchMedia("(pointer: coarse)")` (`viewport.ts`); it only changes the instruction text and button sizes (L207, L316-323).
- Entering annotate mode puts a full-page overlay up, locks body scroll, and sets `touch-action: none`. The overlay listens to both mouse and `touchstart`/`touchmove`/`touchend` (L344-351); `touchstart` calls `preventDefault`.
- On release, if the gesture is under 10 px on either axis it is a **tap**: the overlay's pointer events are switched off for one `elementFromPoint` call (drilling into open shadow roots), an SVG inner node is lifted to the element hosting the `<svg>`, and that element is annotated with a full-bounds rect `{0,0,1,1}` (L16, L646-685, L520-557). A larger gesture is a drawn rectangle.
- Same code path serves a mouse click and keyboard Enter on the focused element.
- Optional right-click / long-press entry (`enableRightClickComment`) relies on `contextmenu`, which iOS never fires for long-press (`types.ts` L298-302), so iOS users must start from the floating button.
- There is no hover-highlight-before-pick step on any device: you tap or draw, then see the highlight with the comment form.

## 7. Project health

Observed 2026-11-17.

- **Age**: repo created 2026-04-02; first npm publish 2026-04-03. About 7.5 months old.
- **Commits**: 450 on `main`. By month: Apr 110, May 93, Jun 12, Jul 51, Aug 5, Sep 56, Oct 86, Nov 37 (to the 16th). 116 in the last 30 days.
- **Maintainers**: effectively one. `NeosiaNexus` has 346 of 450 commits; the next human contributors have 11, 7, 2 and 1. Bots account for about 79.
- **Releases**: automated with release-please, per package. `@siteping/widget` has 38 published versions; five in the first half of November 2026. `@siteping/server` was first published 2026-10-05 and has 8 versions. 157 git tags.
- **Versioning**: everything is 0.x. `SECURITY.md` supports only the latest 0.x. The HTTP protocol is versioned separately (`SITEPING_PROTOCOL_VERSION = 1`, `types.ts` L1458) and described by a checked-in OpenAPI file with a snapshot test.
- **Popularity**: 86 stars, 26 forks, 53 open issues (GitHub metadata as rendered by a fetch tool; the count may include pull requests).
- **npm downloads, 2026-10-18 to 2026-11-16**: `@siteping/widget` 62,049; `@siteping/server` 13,769 (`api.npmjs.org/downloads/point/last-month/...`).
- **Licence**: MIT (root `package.json`, each package, `LICENSE`).
- **Engineering signals**: conformance kit for stores, API-surface snapshot files (`api/*.api.md`), Playwright e2e including a mobile spec, size budgets, CodeQL and OpenSSF badges in the README.

## Implications for hover-note

**Adapter feasibility verdict: feasible, as a translating adapter, not an opaque pass-through.** The Thread, lifecycle and auth parts map cleanly. The Anchor part does not: SitePing's anchor is a fixed, validated, column-typed shape for "one element plus a rect", and hover-note must either speak it or give it up.

Recommended shape (**Inference**): hover-note's Backend interface should treat the Anchor as something a Backend may *transform*, not just store. The SitePing adapter maps the Anchor into `cssSelector` / `textSnippet` / prefix / suffix so the dashboard stays useful, and the adapter documents that text-range detail is lost unless smuggled.

Risks:

- **Single maintainer, 0.x, very high churn.** `@siteping/server` is six weeks old. The wire shape is protected by a protocol version and OpenAPI snapshot, which helps, but nothing is promised before 1.0.
- **Anchor fidelity.** No text ranges, no opaque field. Smuggling JSON into `xpath` works today by reading the code but depends on an unenforced gap.
- **Identity.** A feedback needs a valid email and has no notion of an agent author. Agents writing Comments will appear as `client` unless they hold the API key or the host writes an `access` policy.
- **First Comment is special.** It lives in `message`, cannot be edited or removed alone, and a `type` must be invented.
- **No sync primitive.** Poll-only, no "changed since", comments embedded whole in list responses.
- **Public POST by default.** With `apiKey`, creating feedback and comments is unauthenticated and rate limiting is left to the host. hover-note inherits that exposure when pointed at such a server.
- **CORS is off unless `allowedOrigins` is set**, and it is exact-match only; a hover-note widget on another origin needs the host to list it.

Open questions:

1. Does hover-note's Anchor need text ranges at v1? If yes, is the lossy mapping acceptable for the SitePing Backend, or do we ask upstream for an opaque `meta` JSON field on annotations?
2. Should hover-note's own Anchor fields be chosen to line up with SitePing's (selector, text quote, prefix, suffix, optional host key) so the mapping is near one-to-one?
3. Should the adapter talk HTTP (works from the browser, subject to CORS and the public-POST model) or wrap a `SitepingStore` server-side (full control, but then hover-note needs a server piece)?
4. How do agents authenticate: the shared `apiKey` (grants status changes, deletes and `team` role all at once) or a host-written `access` policy?
5. Is an empty-`annotations` feedback acceptable as a fallback for Anchors that cannot be mapped?

Not verified:

- No live request was made. Empty `annotations`, unknown-key stripping and JSON-in-`xpath` are read from the schema, not tested.
- Bundle sizes are the repo's budgets, not a measurement.
- Open-issue count and stars come from a third-party rendering of GitHub metadata; the GitHub API was not reachable from this session. Issue contents were not read.
- The Drizzle store, the dashboard's data layer (`use-inbox.ts`) and the docs site beyond its landing page were only skimmed.
