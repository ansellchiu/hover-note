# Market landscape: tools that annotate live web pages for humans and agents

Research date: 2026-11-17. Ticket: "Which existing tools annotate live web pages for humans and agents, and what do they leave open?"

## How to read this

- **Sourced fact** is cited inline with a URL. **Synthesis** is confined to the sections marked "Synthesis".
- Retrieval: Exa, Tavily and Parallel search/fetch, plus direct reads of the npm registry (`registry.npmjs.org`) for versions, dates and licences.
- **Bundle sizes marked "measured"** are my own measurement: `gzip -9` of the main browser file in the latest published npm tarball. These are upper bounds, not app-bundle costs: several packages ship unminified or include CSS in JS, and none were tree-shaken. bundlephobia and the npm downloads API were unreachable from this environment.
- Stars are as rendered on 2026-11-17 by the fetch tool, not from the GitHub API (which was not accessible here).
- `?` means not verified in this pass. It does not mean "no".

Confidence overall: **high** on what each open-source tool does (READMEs and docs read directly); **medium** on SaaS widget internals (pricing pages read, anchoring internals mostly not); **low** on download counts.

## 1. Agent-facing tools

| Tool | Targets | Anchor | Data lives | Threads / resolve | Agent access | Framework | Size (gz) | Touch | Licence / price | Traction |
|---|---|---|---|---|---|---|---|---|---|---|
| **Agentation** | element, text, multi-select, area | CSS selector/element path, text, position; React component tree | `localStorage` (cleared after 7 days); optional local server (SQLite) | Yes with server: acknowledge / resolve / dismiss / reply | Read: clipboard, MCP. Write: MCP reply/resolve/dismiss. Cannot create | React 18+ only | ~136 KB measured (likely unminified) | No ("mobile not supported") | PolyForm Shield 1.0.0, free | 5,406 stars; v3.1.2, 2026-09-22 |
| **React Grab** | element (hover + Cmd/Ctrl+C) | none stored; copies HTML + component stack + `file:line` | clipboard only | No | Read: clipboard paste | React | ~99 KB measured (`index.global.js`) | ? | MIT, free | 7,710 stars; v0.2.0, 2026-08-16 |
| **stagewise** | element (in its own browser) | n/a (session context) | in-app session | No | Its own built-in agent | Any site, but inside the stagewise IDE | n/a (desktop app) | n/a | AGPLv3; BYO keys | 6,835 stars; old `@stagewise/toolbar` last published 2025-08-04 |
| **Onlook** | element on a canvas; "spatial comments" | code instrumentation mapping DOM to source | Onlook project (Supabase); hosted | Comments exist; lifecycle ? | Built-in AI chat | Next.js + Tailwind focus | n/a (app) | ? | Apache-2.0; hosted product in early access | 27,219 stars |
| **click-to-component** | element (Option+click) | React fiber `_debugSource` | none | No | None (opens editor) | React | small (few KB) | No (modifier key) | MIT/ISC, free | 2,352 stars; v1.1.3, 2025-09-23 |
| **LocatorJS** | element (click) | build-time source data | none | No | None (opens editor) | React, Preact, Solid, Svelte, Vue | ? | No | MIT, free | 1,828 stars; runtime v2.0.0, 2026-11-01 |
| **Vercel Toolbar comments** | point on page, text highlight | ? (markdown export includes DOM selector + React tree, per a user report) | Vercel SaaS | Yes: threads, resolve/reopen, inbox | Read + write via `vercel comments` CLI (`--format json`) and Vercel MCP (list/get/reply/resolve/edit/react). Create only in the toolbar | Any HTML site; needs Vercel project + account | ? | ? | Proprietary; free on all plans | n/a |
| **Cursor Design Mode** | element, multi-element, area + markup, text edit | xpath, component, attributes, computed styles, fiber props + screenshot | sent to agent; not persisted as threads | No | Cursor's agent only | Any app in Cursor's browser | n/a | n/a | Proprietary (Cursor) | n/a |
| **Windsurf Previews** | element ("Send element") | ? | inserted into Cascade prompt as @-mention | No | Cascade only | "most web projects, React or otherwise" | n/a | n/a | Proprietary | n/a |
| **Claude Code artifact comments** | text highlight on a published artifact page | ? | claude.ai | Yes: threads, resolve, reopen | Claude reads; replies/resolves only threads a person "activated" | Only artifacts Claude published, not your running app | n/a | ? | Team/Enterprise sharing | n/a |
| **lavish-axi** | element, text range, table cell, diagram node | CSS locator + range anchors + snapshot | local CLI server; unsent queue in `sessionStorage` | Conversation per session, no resolve lifecycle | Read: `lavish-axi poll` (CLI). Write: `reply` | Agent-generated HTML files in a sandboxed iframe, not your app | n/a (CLI) | ? | MIT, free | 4,565 stars; v0.1.97, 2026-11-13 |
| **Drafty** | element; point on screenshot for live sites | server re-anchors per pushed version; element box + viewport + revision | drafty.im SaaS | Yes: threads, resolve/reopen, live cursors | Read + write via `drafty comments` CLI (ls/inbox/watch/reply/resolve) | Pushed HTML/Markdown, or screenshots of a live URL; not an in-page widget | n/a | Yes (phone) | Plugin repo public; hosted service; price ? | ? |
| **Chrome DevTools MCP** | none (no annotation concept) | n/a | n/a | No | Agent drives/inspects Chrome | Any | n/a | n/a | Apache-2.0 | 57,819 stars; v1.13.0, 2026-11-11 |
| **v0 Design mode** | element (hover + click) | ? | pending edits until Apply, then a chat version | No | v0 only | v0 projects | n/a | No ("not available on mobile viewports") | Proprietary | n/a |
| **Lovable preview toolbar** | element(s), inline text, drawn annotation, **comment** | ? | Lovable SaaS | Comment threads; lifecycle ? | Lovable only (thread can be sent to it) | Lovable projects | n/a | ? | Proprietary; comments free | n/a |
| **Bolt "Select"** | element | ? | chat prompt | No | Bolt only | Bolt projects | n/a | ? | Proprietary | n/a |

### Discovered, not in the ticket (these matter most to the verdict)

| Tool | Targets | Anchor | Data lives | Threads / resolve | Agent access | Framework | Size (gz) | Touch | Licence / price | Traction |
|---|---|---|---|---|---|---|---|---|---|---|
| **review-loop** | text selection, element (Alt+click), page note | XPath ranges + surrounding context; CSS selectors for elements | **`inline-review.json` in the project root; "commit it for shared review"** | Status open / in_progress / addressed + agent reply; reviewer Accept or Reopen | MCP (stdio): `list_annotations`, `start_work`, `finish_work`; REST API | Astro, Vite, Express/Connect (server middleware required) | ~23 KB measured (`client.js`) | No (Alt+click; no touch/pointer listeners found in dist) | MIT, free | **1 star**; v0.9.0, 2026-03-13 |
| **ui-review** | element, rectangle area | selector, DOM path, text, a11y, layout; missing-target detection + re-anchoring | **`.ui-review/events.jsonl` (append-only, local)** | Yes: threaded human + agent replies; open / in_progress / review / resolved | MCP + Claude Code skills; atomic claims for parallel agents | Any app, via an HTTP/WebSocket **proxy** | n/a (proxy-injected) | ? (has responsive UI) | MIT, free | **2 stars**; repo created 2026-07-16 |
| **instruckt** | element, screenshot region | selector + framework component detection | `localStorage`; with Vite plugin, JSON + screenshots in **`.instruckt/`** on disk | Dismiss state; no reply threads seen | Clipboard markdown; MCP only via Laravel or Tauri backends | "Framework-agnostic JS core" + Livewire/Vue/Svelte/React adapters | ~35 KB measured (IIFE) | ? (touch listeners present in bundle) | MIT (npm), free | 66 stars; v0.4.32, 2026-03-25 |
| **agent-ui-annotation** | element | selector + component path | `localStorage` (7 days) | No | Clipboard markdown | Web Component: vanilla, React, Vue, Svelte, Angular | ~42 KB measured | ? | MIT, free | 53 stars; v0.8.0, 2026-08-18 |
| **astrotation** | element (hover, arrow keys for parent/child) | Astro `file:line` + selector | `.astrotation/annotations.json` (README says gitignore it) | Yes: thread, pending / in progress / feedback / resolved / dismissed | MCP: list, get, watch, acknowledge, feedback, dismiss, reply, clear | Astro only | ~9 KB measured (toolbar source) | ? | MIT, free | 0 stars; v0.5.0, 2026-07-10 |
| **onUI** | element, multi-select, rectangle/ellipse region | selector/path | local store via browser native messaging | Status metadata; no reply threads seen | MCP: 8 tools incl. update/delete | Any page (browser **extension**) | n/a | No (desktop extension) | GPL-3.0, free | 108 stars; v2.2.3, 2026-03-15 |
| **pinthread** | element, area | selector + visible text + position | pinthread.dev SaaS | Yes: replies, reactions, resolve/reopen | Remote MCP with read or write tokens (list, get, reply, start, resolve, reopen); copy-all; webhooks | Any site, one `<script>` tag | ? | ? | Proprietary, free | ? |
| **MarkLayer** | 16 tools: comment pin, text selection, area, inspector, drawing | selector, text fingerprint, React/Vue component | marklayer.app (share link); proxy or extension | Yes: 5 statuses, priority, assignee (person or agent) | MCP (`marklayer-mcp`): watch, resolve, reply; vendor says agents can also leave annotations | Any URL via proxy/extension | n/a | ? | Free; open-source status ? | ? |
| **Caliper** | element, lasso | stable selector + component + token-matched styles | export file / extension | Agent-to-human questions pinned to elements | MCP both directions | Any page (Chrome extension) | n/a | No | MIT | 3 stars |

Other clones seen but not analysed: agentation-svelte, agentation-vue, annotaKit (Svelte), annotaitr, annotron, anriss (Rust + SQLite + MCP, W3C Web Annotation model).

### Notes and sources (agent-facing)

- **Agentation.** React 18+ peer dependency, "Desktop browser (mobile not supported)", PolyForm Shield licence: https://github.com/benjitaylor/agentation . Storage ("`localStorage`, keyed by page pathname... cleared after 7 days"; with Agent Sync "synchronized to the MCP server"): https://agentation.com/faq . Nine MCP tools, all read/ack/resolve/dismiss/reply/watch, none that create an annotation: https://agentation.com/mcp . `agentation-mcp` depends on `better-sqlite3`: https://www.npmjs.com/package/agentation-mcp . The npm page as rendered to me reported 2,069,057 weekly downloads for `agentation` and 152,501 for `agentation-mcp`; I could not cross-check against the npm downloads API and the ratio to stars looks unusual, so treat as unverified.
- **React Grab.** Hover + copy, output is element HTML plus component stack with source location; exposes `react-grab/primitives` for building custom pickers: https://github.com/aidenybai/react-grab . No persistence, comment or MCP feature appears in the README or site (https://www.react-grab.com/); I did not audit the source for one.
- **stagewise.** Now "The Open Source Agentic IDE"; AGPLv3; stagewise account service "will be sunset at the end of November 2026": https://github.com/stagewise-io/stagewise . The in-page toolbar npm package was last published 2025-08-04 (registry read). It has left the embeddable-toolbar category.
- **Onlook.** Architecture and Next.js/Tailwind focus: https://github.com/onlook-dev/onlook . "Share your canvas, leave spatial comments" appears in its features page source: https://github.com/onlook-dev/onlook/blob/a242be58/apps/web/client/src/app/features/page.tsx . The README now points to a waitlist for a new hosted design tool.
- **click-to-component / LocatorJS.** Source-jump only: https://github.com/ericclemmons/click-to-component , https://github.com/infi-pc/locatorjs .
- **Vercel.** Comments require a Vercel account; "click on the page or highlight text": https://vercel.com/docs/comments . Toolbar works on localhost and production via `@vercel/toolbar`: https://vercel.com/docs/vercel-toolbar/in-production-and-localhost . CLI: list, inspect, reply, resolve, reopen, update, delete; "Use the Vercel Toolbar to create new comment threads": https://vercel.com/docs/cli/comments and https://vercel.com/changelog/manage-vercel-toolbar-comments-from-the-cli . MCP tool names (`list_toolbar_threads`, `reply_to_toolbar_thread`, `change_toolbar_thread_resolve_status`...) are from a third-party catalogue, not Vercel's docs: https://neuraltrust.ai/mcp-catalog/vercel .
- **Cursor.** https://cursor.com/docs/agent/design-mode and https://cursor.com/blog/design-mode (2026-06-05).
- **Windsurf.** https://docs.windsurf.com/windsurf/previews (same page states "The Windsurf Browser has been deprecated").
- **Claude Code artifact comments.** https://code.claude.com/docs/en/artifacts ("Collect comments on an artifact"). Comments apply to org-shared artifacts; public-link viewers cannot comment. Text-highlight targeting is from a secondary write-up (https://techhacks.io/ai/claude-code-artifact-comments/), not the official page. See also the sibling note `docs/research/claude-artifact-comments.md`, which I did not read or modify.
- **lavish-axi.** https://github.com/kunchenguid/lavish-axi . Reviews agent-generated HTML files; not a layer on your own running app.
- **Drafty.** https://github.com/drafty-im/drafty , https://drafty.im/kb/how-anchored-comments-work , https://drafty.im/kb/annotate-live-website (live sites become screenshot boards with point anchors, "nothing is installed on the site itself"). Pricing page did not load.
- **Chrome DevTools MCP.** https://github.com/ChromeDevTools/chrome-devtools-mcp . Complementary, not competing: it gives an agent eyes, not a place to leave notes.
- **v0 / Lovable / Bolt.** https://v0.app/docs/design-mode , https://docs.lovable.dev/features/preview-toolbar , https://support.bolt.new/building/using-bolt/interacting-ai .
- **review-loop.** https://github.com/viv/review-loop .
- **ui-review.** https://github.com/flucas96/ui-review . An npm package named `ui-review` exists (v0.1.9, 2026-04-28) but predates this repo's creation date, so it may be unrelated; not verified.
- **instruckt.** https://github.com/joshcirre/instruckt .
- **agent-ui-annotation.** https://github.com/YeomansIII/agent-ui-annotation .
- **astrotation.** https://github.com/dpush91/astrotation .
- **onUI.** https://github.com/onllm-dev/onUI , https://github.com/onllm-dev/onUI/blob/main/docs/mcp-setup.md .
- **pinthread.** https://pinthread.dev/ , https://pinthread.dev/agents .
- **MarkLayer.** https://marklayer.app/ (vendor's own claims, including its comparison of competitors).
- **Caliper.** https://github.com/DenDiem/caliper .

## 2. Human feedback widgets

| Tool | Targets | Anchor | Data lives | Threads / resolve | Agent access | Framework | Size (gz) | Touch | Licence / price | Traction |
|---|---|---|---|---|---|---|---|---|---|---|
| **SitePing** | rectangle, element click/tap, right-click point | Multi-signal: CSS selector, XPath, text + context, structural fingerprint, optional `data-feedback-anchor`; scored re-resolution with confidence; orphans hidden | **Self-hosted DB** (Prisma or Drizzle); localStorage/memory adapters; adapter kit | Yes: threads; open / in_progress / resolved / wont_fix | "Plain HTTP API"; "panel actions" to hand a feedback to an agent; GitHub/GitLab issue sync; webhooks. **No MCP server found** | Framework-agnostic + React hook | ~51-59 KB first load (vendor); ~40 KB entry measured | **Yes** (tap an element; bottom sheets on phones) | MIT, free | 86 stars; 17,913 weekly downloads (npm page); v0.12.5, 2026-11-16 |
| **Marker.io** | screenshot + annotation | ? | SaaS | Issues with status + comments | Hosted MCP (since 2026-03-18): read, set resolved/archived, comment; all plans | Any (snippet/extension) | ? | ? | From $39/mo yearly, $59 monthly | n/a |
| **BugHerd** | pinned element/point | CSS selector captured (secondary source) | SaaS | Tasks with status + comments | Hosted MCP (beta, free for all users): list/create/update tasks | Any | ? | ? | From $42/mo yearly, $50 monthly | n/a |
| **Userback** | screenshot annotation, video, session replay | ? | SaaS | Feedback items + comments + workflow | "MCP connectors"; REST API on top tier | Any | ? | ? | Free plan; paid from $39-49/mo | n/a |
| **Markup.io** | comments on proxied pages/files | ? | SaaS | ? | API/SDK listed; no MCP found | Any URL | n/a | ? | Pro from $79/mo, no free plan | n/a |
| **Pastel** | click-to-comment on proxied live site | ? | SaaS | Comments, resolve, export | No MCP found; webhooks + PM integrations | Any URL (proxy) | n/a | ? | Free (1 canvas); Pro $35/mo; Enterprise $450/mo | "12 million comments" (vendor) |
| **Ybug** | screenshot annotation, video | ? | SaaS | Feedback + replies (Startup tier) | MCP (beta): read-only on Basic, write on Startup; REST API | Any (snippet) | ? | ? | Free (1 project); Basic EUR 10-13/mo; Startup EUR 23-29/mo | "1,000+ teams" (vendor) |
| **Feedbucket** | screenshot + video feedback | ? | SaaS | Feedback collaboration | "MCP Connector" + API on all plans | Any (snippet) | ? | ? | Pro $49/mo; Business $89/mo | n/a |
| **Jam** | recording (not element-pinned) | n/a | SaaS | n/a (links between Jams) | Jam MCP reads a Jam; Jam CLI lets an agent record one | Browser extension | n/a | Mobile app exists (secondary) | Free tier; paid ? | n/a |

Sources: SitePing https://github.com/NeosiaNexus/SitePing , https://siteping.dev/docs , https://siteping.dev/docs/widget/anchoring , https://www.npmjs.com/package/@siteping/widget . Marker.io https://marker.io/mcp-use-case-library , https://help.marker.io/en/articles/14034657-mcp-integration-model-context-protocol , https://marker.io/blog-categories/product-updates . BugHerd https://bugherd.com/feature/mcp , https://bugherd.com/feature/ai . Marker.io and BugHerd prices are as quoted in SitePing's README ("checked on 2026-10-30"), not read from the vendors' pricing pages. Userback https://userback.io/pricing , https://docs.userback.io/docs/userback-mcp . Markup.io https://www.markup.io/pricing-page-faqs . Pastel https://usepastel.com/plans and its own blog for the Pro price. Ybug https://ybug.io/pricing . Feedbucket https://www.feedbucket.app/pricing . Jam https://jam.dev/docs/jam-mcp . BugHerd's MCP write scope and CSS-selector capture are from a competitor's comparison (https://marklayer.app/guides/bug-report-mcp-servers), so medium confidence.

Fact worth stating plainly: **every SaaS widget in the ticket's list except Markup.io and Pastel now advertises an MCP connector.** Agent read access to human feedback stopped being a differentiator during 2026.

## 3. Gap table against hover-note's requirements

Y = yes, N = no, ~ = partial, ? = not verified.

| Tool | Framework-agnostic | ≤10 KB gz | Hover picking + touch | Persistent threads | Git-versioned files | Agent read | Agent write | Self-hostable backend |
|---|---|---|---|---|---|---|---|---|
| **hover-note (target)** | Y | Y | Y | Y | Y (default) | Y | Y | Y |
| review-loop | ~ (Astro/Vite/Express plugin) | N (~23 KB) | ~ hover N/A, Alt+click; no touch | ~ (status + one reply, then Accept deletes) | **Y** (single JSON, optional commit) | Y (MCP) | ~ (status + reply; cannot create) | Y (it is a local file) |
| ui-review | Y (proxy) | n/a | ~ / ? | **Y** | ~ (local `.ui-review/events.jsonl`; committing not documented) | Y (MCP) | Y (reply, status) | Y (local) |
| Agentation | N (React) | N | hover Y, touch N | ~ (7-day localStorage, or local SQLite server) | N (issue #66 open) | Y | ~ (reply/resolve; cannot create) | ~ (local server) |
| instruckt | Y | N (~35 KB) | hover Y, touch ? | N | ~ (`.instruckt/` on disk) | ~ (clipboard; MCP on Laravel/Tauri) | N / ? | ~ |
| astrotation | N (Astro) | ~ (~9 KB unminified source) | hover Y, touch ? | Y | ~ (JSON file, recommended gitignored) | Y | Y (reply/feedback) | Y (local) |
| agent-ui-annotation | Y | N (~42 KB) | hover Y, touch ? | N | N | ~ (clipboard) | N | N |
| React Grab | N (React) | N | hover Y, touch ? | N | N | ~ (clipboard) | N | N |
| SitePing | Y | N (~51-59 KB) | **Y** (tap) | **Y** | N (database) | ~ (HTTP API; no MCP found) | ~ (HTTP API) | **Y** |
| pinthread | Y (script tag) | ? | ? | Y | N (SaaS; JSON download) | Y (MCP) | Y (reply/resolve) | N |
| Vercel comments | ~ (needs Vercel) | ? | ? | Y | N | Y (CLI, MCP) | Y (reply/resolve; cannot create) | N |
| MarkLayer | Y (proxy/extension) | n/a | ? | Y | N | Y | Y (incl. create, per vendor) | N / ? |
| Drafty | n/a (hosted canvas) | n/a | Y | Y | N (versioned server-side) | Y (CLI) | Y | N |
| onUI | Y (extension) | n/a | hover Y, touch N | ~ | N | Y | ~ | Y (local) |
| Marker.io / BugHerd / Ybug / Feedbucket / Userback | Y | ? | ? | Y | N | Y (MCP) | ~ | N |
| Cursor / Windsurf / v0 / Lovable / Bolt pickers | N (locked to host product) | n/a | hover Y, touch N or ? | N (Lovable: comments) | N | host agent only | host agent only | N |
| Claude Code artifact comments | N (artifacts only) | n/a | ? | Y | N | Y (Claude only) | ~ (activated threads) | N |

No row other than the target has Y in every column. No surveyed tool has both "≤10 KB" and "persistent threads". No agent-facing in-page tool has verified touch Targeting.

## 4. The three closest competitors (Synthesis)

### 4.1 review-loop: closest in architecture

It already does the core idea: a dev-only overlay, text-range and element Targeting, a JSON file in the repo that the README says to commit for shared review, and an MCP server over that same file with a status lifecycle and agent replies.

hover-note would differ by:
1. **No server integration.** review-loop needs an Astro/Vite/Express plugin to serve its client and REST API. hover-note's "one script tag" promise means the file Backend needs a different write path (see open question below).
2. **Size.** ~23 KB gzipped client against a 10 KB budget.
3. **Touch.** Alt+click cannot be done on a phone; hover-note specifies a touch equivalent.
4. **Thread model.** review-loop is open → in_progress → addressed, then Accept *deletes* the annotation. hover-note's Thread is an ordered list of Comments with an open/resolved lifecycle that stays in history.
5. **Agent-authored Annotations.** review-loop's three MCP tools cannot create one.
6. **Pluggable Backends.** review-loop has one storage path.
7. **Git ergonomics.** One mutable JSON file shared by the browser UI, REST API and MCP server will produce merge conflicts across branches. A file-per-Annotation or append-only layout is an obvious improvement (ui-review already uses append-only JSONL).

Honest caveat: review-loop has 1 star and no release since March 2026. It is evidence that the architecture works, and also evidence that the architecture alone did not attract users.

### 4.2 Agentation: closest in mindshare

It owns the phrase "visual feedback for agents" (5.4k stars, many ports). It has threads via MCP (acknowledge, resolve, dismiss, reply).

hover-note would differ by:
1. **Framework.** Agentation requires React 18+. The Svelte request (#39) is open since January 2026 and has spawned at least four third-party ports.
2. **Persistence.** localStorage with 7-day expiry, or a local SQLite server whose webhook deliveries are "best effort and in memory". Nothing lands in git. Issue #66 asks for exactly that and is still open.
3. **Licence.** PolyForm Shield is source-available with a non-compete clause, not OSI open source. hover-note being MIT/Apache is a real difference for adopters and for anyone wanting to embed it.
4. **Touch.** Explicitly unsupported.
5. **Size.** Roughly an order of magnitude over 10 KB even allowing for my measurement being pessimistic.
6. **Agent write.** Agents can reply and resolve but not open a new Annotation.

Risk: the maintainer commented on #66 in September 2026 that portable import/export "needs a versioned format and data-merge rules" and is being kept open. If Agentation ships file export, differentiator 2 shrinks to "default vs option".

### 4.3 SitePing: closest on Anchor quality, touch and self-hosting

It has the strongest documented Anchor design of anything surveyed (multi-signal, scored, confidence shown, orphans hidden rather than misplaced), real touch support, threads with a lifecycle, MIT licence, an adapter kit, and active releases (v0.12.5 the day before this research). `CONTEXT.md` already names it as a possible remote Backend.

hover-note would differ by:
1. **Audience.** SitePing is "client feedback" for freelancers and agencies: reviewer identity (name + email), dashboard inbox, webhooks. hover-note is developer-and-agent first.
2. **Default Backend.** SitePing needs an HTTP endpoint and a database for anything shared (localStorage adapter is single-browser). hover-note defaults to files in git, with no service to run.
3. **Agent access.** No MCP server or agent CLI found; agents would use the HTTP API or issue sync. hover-note makes agent read and write first-class.
4. **Size.** ~51-59 KB first load against 10 KB.
5. **Targeting gesture.** SitePing leads with drawing a rectangle; hover-note leads with hover element picking.

Because SitePing has an adapter contract and a conformance suite, the cheaper path for some of hover-note's goals may be to *be* a SitePing-compatible client or to reuse its anchoring ideas, rather than compete with it. Its anchoring page is worth reading before designing the Anchor.

Runner-up: **ui-review** is the closest on Thread semantics (append-only JSONL, threaded human + agent replies, re-anchoring, atomic claims for parallel agents) but is a proxy-based local tool with 2 stars.

## 5. Verdict (Synthesis)

**The gap is real but narrow, and it is a combination gap rather than a capability gap.**

What is already filled:
- Point at an element and tell an agent: saturated. Agentation, React Grab, every IDE and every app builder.
- Persistent threads that an agent can read, reply to and resolve: filled several times over in 2026 (Agentation + MCP, Vercel CLI/MCP, pinthread, MarkLayer, Drafty, ui-review, astrotation, and most SaaS widgets via MCP).
- Annotations as files in the repo read by an agent over MCP: exists (review-loop, ui-review, instruckt, astrotation), all tiny projects.
- Self-hosted, MIT, robust anchoring, touch, threads: SitePing.

What nobody surveyed has:
- All eight requirements together. The nearest (review-loop) misses on size, touch, framework setup and Thread model.
- An in-page annotator with persistent threads at or under 10 KB gzipped. The smallest threaded widget measured is ~23 KB.
- Touch Targeting in an agent-facing in-page tool. Only SitePing (human-facing) and Drafty (hosted canvas) document it.
- Git as the *default* store with a format designed for merges and history. Existing file-backed tools treat the file as a local scratch store; one even recommends gitignoring it.
- Agents creating Annotations on the user's own running page as first-class participants. Only MarkLayer claims this, as a SaaS.
- A true OSI licence plus pluggable Backends in the agent-facing group's leader (Agentation is PolyForm Shield).

Reasons for caution:
1. **Low traction of the nearest neighbours.** review-loop (1 star), ui-review (2), astrotation (0) already ship "files + MCP + threads". Either distribution was the problem or demand for the git-file property is weak. This research cannot tell which.
2. **Platform absorption.** Cursor, Lovable, Vercel and Claude Code now ship the casual version of this inside products people already use. hover-note's audience is people who are *not* inside one of those, or who use several.
3. **The 10 KB budget fights the feature list.** SitePing's anchoring alone is far bigger than that. Text-range Anchors, scored re-resolution, a Thread UI, touch handling and a Backend interface in 10 KB is an engineering claim no existing tool supports. It is the least-proven requirement.
4. **One script tag versus writing files.** A browser cannot write to the repo. Every file-backed tool surveyed uses a dev-server plugin, proxy or local daemon. hover-note needs an answer (dev-server middleware, a tiny local CLI, File System Access API, or export-only) and that answer decides how much of "one script tag" survives.

Net: worth building if the thesis is "git-native, tiny, touch-capable, truly open, both-direction" and the project accepts it is entering a crowded field as a differentiated small library, not opening a new category. Not worth building if the thesis is only "agents can read page annotations", which is now table stakes.

## 6. Signals of demand

Direct evidence found:

- **Files in version control.** Agentation issue #66, "File export for team workflows" (opened 2026-02-01, still open): asks to export annotations to a `.agentation/` directory "that can be committed to version control" for team sharing, history and status visibility. Maintainer reply 2026-09-21 keeps it open pending a versioned format. https://github.com/benjitaylor/agentation/issues/66
- **Multi-user sharing.** Agentation #116, real-time multi-user sync over SSE, from a user embedding Agentation in a diff viewer. https://github.com/benjitaylor/agentation/issues/116
- **Framework-agnostic.** Agentation #39, Svelte/SvelteKit support (open since 2026-01-23), with a commenter asking for a browser extension instead and another having an agent write a wrapper. https://github.com/benjitaylor/agentation/issues/39 . Third-party ports: https://github.com/Frank-III/agentation-svelte , https://github.com/Blaked84/agentation-vue , https://github.com/YeomansIII/agent-ui-annotation , and annotaKit, whose Reddit thread includes the question "Does anyone know about a good one that works in a vanilla JS context?" https://www.reddit.com/r/sveltejs/comments/1rlh9id/nextjs_has_agentation_so_i_built_annotakit_for
- **Agent access to existing comment threads.** Vercel Community thread asking for preview comments to be exposed to AI agents ("Sync Vercel preview deployment comments to GitHub PR for AI agent feedback loops"), with several users adding the same request; Vercel later shipped the `vercel comments` CLI with a ready-made agent prompt. https://community.vercel.com/t/sync-vercel-preview-deployment-comments-to-github-pr-for-ai-agent-feedback-loops/31663 , https://vercel.com/changelog/manage-vercel-toolbar-comments-from-the-cli
- **Supply-side signal.** At least eight independent projects built some version of "annotate page, agent reads via MCP" between January and July 2026 (section 1). Many builders had the itch; few of the results found users.
- **Vendor convergence.** Marker.io, BugHerd, Userback, Ybug, Feedbucket and Jam all added MCP in 2026 (section 2).

Weak or absent evidence:

- **Touch / mobile.** Agentation and its Svelte port both state mobile is unsupported, but I did not find an issue asking for it. Demand for touch Targeting in the agent-facing group is unproven. SitePing treats it as a headline feature for client review.
- **Bundle size.** I found no user asking for a smaller annotator. The 10 KB requirement is a design preference, not an observed demand.
- **Agent-authored Annotations.** Only supply-side (Caliper's agent-to-human questions, MarkLayer). No user requests found.
- Hacker News: my searches returned nothing relevant; this is a search miss, not evidence of absence.

## 7. Could not verify

- npm weekly downloads for anything except the three figures quoted (downloads API blocked); bundlephobia figures (blocked). Bundle sizes here are my own tarball measurements.
- GitHub stars via API, last-commit dates, and issue counts beyond what the fetch tool rendered.
- Anchor internals of Marker.io, BugHerd, Userback, Markup.io, Pastel, Ybug, Feedbucket, pinthread, Vercel and Lovable.
- Touch support for React Grab, instruckt, agent-ui-annotation, ui-review, pinthread, MarkLayer.
- Whether MarkLayer's client/server is open source; Drafty's and Jam's pricing; Onlook's comment lifecycle and current open-source release cadence.
- Vercel's MCP toolbar tool list against Vercel's own docs.
- Whether React Grab has added comment or MCP features beyond what its README shows.
