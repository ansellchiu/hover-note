# lavish-axi: how it picks elements, anchors feedback and hands it to the agent

Research note, 2026-11-17.

**Sources.** Source code read, never run:

- `kunchenguid/lavish-axi` at commit `d0c1667` (2026-11-16), package version 0.1.97, MIT, "Copyright (c) 2026 Kun Chen". Paths below are relative to that repo.
- `kunchenguid/axi` at commit `c76bcf2` (2026-11-16), MIT, same copyright holder.

Claims marked **Observed** are read from the cited lines. Claims marked **Inference** are my reading of what the code implies; nothing was executed, so no behaviour was confirmed in a browser.

## Summary

lavish-axi is a review loop for one person, one agent and one local HTML file. Its own vision document states this scope (`VISION.md`, "Scope"). The artifact is served in a sandboxed iframe with one injected script. A click or a text selection opens a small card. The note is queued in the browser tab, sent to a local server, and handed to the agent when `lavish-axi poll <file>` returns.

For hover-note the useful parts are small and easy to rewrite: the capture-phase event wiring, the uniqueness-checked CSS path, the text-range target shape, and the shape of the agent-facing output. The parts hover-note needs most are absent: an anchor that survives page change, threads, a resolve state, storage in the repo, and a real touch targeting mode.

## 1. Picking elements and text ranges

All of this lives in `src/artifact-sdk.js`.

**Events (Observed).** Four capture-phase listeners on `document`:

| Event | Role | Lines |
| --- | --- | --- |
| `mouseover` | outline the element under the cursor | 2668-2685 |
| `mouseout` | clear the outline | 2687-2696 |
| `mouseup` | if there is a non-collapsed selection, open the card for a text range and set `ignoreNextClick` | 2698-2716 |
| `click` | `preventDefault` + `stopPropagation`, then open the card for `event.target` | 2718-2756 |

There are no `pointer*` or `touch*` listeners for annotation. Pointer events appear only in the Mermaid pan/zoom helper (999-1003).

**What is skipped (Observed).** Every handler returns early for:

- Lavish's own UI (`[data-lavish-ui]`, 1216-1218);
- elements the page author marked `[data-lavish-action]` (1220-1222);
- native interactive controls and ARIA widgets, via `isNativeInteractiveControl` (117-130): `button, input, select, textarea, option, optgroup, label, summary, [contenteditable]` plus roles such as `button`, `checkbox`, `tab`.

So form controls keep working while annotation mode is on. Links are annotated on a plain click and followed on Cmd/Ctrl+click (2733-2737).

**Mode (Observed).** Annotation is a mode, on by default, switched off with `data-lavish-annotate="off"` on `<html>` (477) or toggled with Cmd/Ctrl+I (9-14, 2643-2651). Entering the mode injects a stylesheet that forces `cursor: default` on everything except controls (1262-1273).

**Highlight drawing (Observed).**

- Elements: the SDK writes inline `outline` and `outline-offset` on the target element itself (1232-1240). Clearing sets `el.style.outline = ""`. There is no overlay box.
- Text ranges: one absolutely positioned `div` per `range.getClientRects()` rectangle, appended to the SDK's shadow root (1247-1260).
- The card and text highlights live in one open shadow root on a `div` appended to `<html>`, `position: fixed; z-index: 2147483647`, with `:host{all:initial}` (2287-2304). Key events are stopped at the shadow root so the page's key handlers do not see typing in the card (2297-2299).
- The card is placed once from the target's `getBoundingClientRect()` and clamped to the viewport (2348, 2400-2405). It does not follow scroll.

**Inference.** Writing inline `outline` mutates the page: it overwrites any inline outline the element already had and does not restore it, and `outline-offset` is never cleared. That is acceptable for a throwaway artifact and wrong for a library dropped into someone's app.

**Parent/child stepping (Observed: absent).** The target is `event.target`, the deepest element under the cursor. There is no keyboard or wheel stepping to a parent or child. The only re-targeting is domain-specific: a click inside a rendered Mermaid node resolves up to the node's `<g>` (915-917), and a click inside a table adds the cell's row and column labels without changing the target (896-900).

**iframes and shadow DOM (Observed).**

- The artifact itself is one iframe with `sandbox="allow-scripts allow-forms allow-popups allow-popups-to-escape-sandbox allow-downloads"` and no `allow-same-origin` (`src/server.js:133`, `:3121`). The SDK talks to the outer page only by `postMessage` (472-474) and accepts messages only from `parent` (2569).
- Inside the artifact, the SDK does not use `composedPath()`, does not read `shadowRoot`, and does not descend into nested iframes. A search of `src/artifact-sdk.js` for `shadowRoot`, `composedPath` and `contentDocument` finds nothing.
- **Inference.** A click inside a page's own shadow tree is retargeted to the shadow host, so the host is what gets annotated. Content in a nested iframe cannot be annotated.

## 2. What is recorded to identify the target

**Element target (Observed).** `context(el)` returns four fields (888-910):

| Field | Content |
| --- | --- |
| `uid` | a per-page-load counter kept in a `WeakMap` (790-793). Not stable across reloads. |
| `selector` | a CSS path, see below |
| `tag` | lowercase tag name |
| `text` | `innerText`, whitespace-collapsed, cut to 240 characters |

**Selector strategy (Observed, 802-832).** Walk up from the element, building `tag` parts joined with ` > `:

- add `#id` (via `CSS.escape`) when the node has an id, and stop climbing only if the path so far matches exactly one element, because "generated pages repeat ids";
- add `:nth-of-type(n)` when the node has same-tag siblings;
- climb at least five levels, then keep climbing until `document.querySelectorAll(path).length === 1` (`selectsOnly`, 876-882).

Classes, `data-*` attributes, roles and ARIA labels are never used.

**Text range target (Observed, 1164-1214).**

```js
{
  type: "text-range",
  text,                       // selection.toString(), whitespace-collapsed, not truncated
  selector,                   // common ancestor element
  commonAncestorSelector,     // same value
  start: { selector, path, offset },
  end:   { selector, path, offset },
}
```

`path` is a list of `childNodes` indexes from the boundary's nearest element down to the text node; `offset` is the DOM range offset. There is no prefix or suffix context around the quote.

**Extra, optional target data (Observed).**

- Table cells: visible row and column labels, omitted when merged cells make them ambiguous (`src/table-cell.js:1-6`, `:110-116`).
- Mermaid nodes: `type`, `diagramId`, `nodeId`, `label`, `selector` (`docs/invariants.md:215`).
- Images: the reviewer may attach PNG, JPEG or WebP files by paste, drop or picker; the agent gets a local file path (`README.md:234-236`).

**Not recorded (Observed).** No `outerHTML` snippet, no coordinates or bounding box, no viewport size, no automatic screenshot, no XPath, no URL or route, no source file or component name. The search of the SDK for `outerHTML`, `html2canvas` and `screenshot` finds only comments about pasted images.

**A whole-page text snapshot is sent alongside (Observed).** `snapshot()` walks the DOM to depth 6 and emits one line per element, `uid=<n> <tag> "<first 80 chars of text>"` (1321-1335). The outer page requests it at send time and the batch carries it as `dom_snapshot`. The prompt's `uid` indexes into this snapshot.

**Re-finding the target (Observed, 834-874).** Within one page load, a queued note is matched to its element by object identity through a `WeakRef`. After a reload it falls back to the stored selector: the same string, or a selector whose single match is the clicked element. If the element is gone, the note is shown as an "Unsent annotation" in the side panel (`README.md:201`). Text-range notes are never re-attached (835).

**Inference.** The anchor is designed to live for one round of feedback. The agent is expected to rewrite the file, after which the notes have been consumed. Nothing here attempts to re-find an element after the page structure changes.

## 3. Storage and delivery to the agent

**Path of one note (Observed).**

1. Card "Queue" calls `queuePrompt`, which posts `lavish:queuePrompt` to the parent window (1280-1311).
2. The outer page keeps the unsent queue in that tab's `sessionStorage` (`src/chrome-client.js:466-482`). `README.md:210`: "Unsent queued notes belong to the tab where you entered them, not to a server-shared draft."
3. "Send to Agent" posts the batch plus the DOM snapshot to `POST /api/:key/prompts`. The route rejects cross-origin requests (`src/server.js:1145-1150`).
4. The server appends to one JSON file, `~/.lavish-axi/state.json`, or `LAVISH_AXI_STATE_DIR` (`src/paths.js:141-145`). The file is rewritten whole under a mutex on every operation (`src/session-store.js:64`, `:817`).
5. `lavish-axi poll <file>` calls `GET /api/poll` and blocks (`src/cli.js:483`). The server answers when a `feedback`, `ended` or `browser-disconnected` event fires for that session (`src/server.js:1098-1110`).
6. `takeFeedback` returns the pending prompts and removes them from `state.json` (`src/session-store.js:620-648`; comment at `src/server.js:567-568`: "`takeFeedback` is destructive"). If the client had already disconnected, the batch is put back (`src/server.js:578-600`).

**Server (Observed).** Express plus `ws`, default port 4387, loopback only unless Tailscale is detected (`README.md:263-264`). One detached background server serves all sessions and stops itself after 30 idle minutes (`README.md:254`).

**Session identity (Observed).** The session key is the first 16 hex characters of the SHA-256 of the canonical file path (`src/session-store.js:826-828`). The agent passes the file path to every command and never sees an ID (`README.md:161`).

**CLI commands (Observed, `README.md:271-287`).**

| Command | Purpose |
| --- | --- |
| `lavish-axi` | list open sessions and usage guidance |
| `lavish-axi <file>` | open or resume a session |
| `lavish-axi poll <file>` | long-poll for feedback; `--agent-reply "..."` posts a reply first |
| `lavish-axi reply <file>` | post a reply without polling |
| `lavish-axi end <file>` | end the session as the agent |
| `export`, `share`, `stop`, `playbook`, `libraries`, `design`, `setup hooks`, `setup plugin`, `server` | not part of the feedback path |

**Long polling details (Observed).**

- One listener per session. A second poll fails with `LISTENER_ACTIVE` unless `--takeover` is passed, in which case the first exits with `LISTENER_REPLACED` (`README.md:308-309`).
- No timeout by default. `--timeout-ms` is described as a "Test/debug escape hatch only" (`README.md:312`).
- Stdout carries only the final response. A "not hung" banner goes to stderr (`src/cli.js:451-456`).
- Four outcomes: `feedback`, `ended`, `browser_disconnected`, `waiting` (`src/cli.js:593-632`).
- The browser shows per-note receipts derived from server-side observation: Seen when a poll takes the note, Working when the file changes, Done when the agent replies (`README.md:219`).

**TOON output (Observed).** The CLI returns a plain object and `axi-sdk-js` renders it with `encode` from `@toon-format/toon` (`axi/packages/axi-sdk-js/src/output.ts:2`, `:57-60`). The object for a feedback poll is built in this key order (`src/cli.js:603-613`): `session`, `prompts`, optional `artifact_failures`, `next_step`, `dom_snapshot`. Each prompt has `uid`, `prompt`, `selector`, `tag`, `text`, and optionally `target` and `attachments` (`src/session-store.js:847-862`). `prompt_id` and `delivery_seq` are stripped before the agent sees them (`src/session-store.js:866-871`, `src/server.js:560-565`).

**Example of what the agent receives.** The repository contains no literal TOON sample, and I did not run the CLI. The block below is reconstructed by hand from the object shape above and the TOON format rules. Field names, order and the `next_step` wording are from the source; exact quoting and indentation are unverified.

```
session:
  file: /home/dev/plan.html
  status: feedback
prompts[2]:
  - uid: "14"
    prompt: Make this heading shorter
    selector: "html > body > main > section:nth-of-type(2) > h2"
    tag: h2
    text: Rollout plan for the third quarter
  - uid: ""
    prompt: This number is wrong
    selector: "html > body > main > section:nth-of-type(2) > p:nth-of-type(1)"
    tag: text
    text: 40% of users
    target:
      type: text-range
      text: 40% of users
      selector: "html > body > main > section:nth-of-type(2) > p:nth-of-type(1)"
      commonAncestorSelector: "html > body > main > section:nth-of-type(2) > p:nth-of-type(1)"
      start:
        selector: "html > body > main > section:nth-of-type(2) > p:nth-of-type(1)"
        path[1]: 0
        offset: 12
      end:
        selector: "html > body > main > section:nth-of-type(2) > p:nth-of-type(1)"
        path[1]: 0
        offset: 24
next_step: "Apply the requested changes to /home/dev/plan.html. Do not respond to the user just yet. If you are continuing to wait for feedback, run `lavish-axi poll /home/dev/plan.html --agent-reply \"<message for the user>\"` without --timeout-ms to reply and keep polling. ..."
dom_snapshot: "uid=1 body \"Rollout plan ...\"\n  uid=2 main ...\n ..."
```

Two points about this output that the source does confirm:

- `next_step` is a long paragraph of instructions that is rebuilt and resent on every feedback poll (`src/cli.js:634-663`).
- `dom_snapshot` comes last on purpose; a test asserts the order prompts, failures, next_step, snapshot (`test/cli-output.test.js:1877-1887`).

## 4. Touch devices

**Observed.**

- The annotation SDK has no touch-specific code. The only match for "touch" in `src/artifact-sdk.js` is `touchAction` on Mermaid SVGs (1009).
- Phone support is in the outer page only. Below 860px the conversation panel becomes a bottom sheet with swipe gestures (`src/chrome-client.js:1295`, `README.md:212`).
- The server can bind to the machine's Tailscale address so a phone on the same tailnet can open the review link (`README.md:264`).
- A phone that locks its screen drops the connection, so the poll would return `browser_disconnected` after 10 seconds unless `LAVISH_AXI_DISCONNECT_GRACE_MS` is raised (`README.md:221-222`).

**Inference.**

- Element annotation works on touch because a tap fires a synthetic `click`, and annotation mode is on by default. There is no hover preview, so the outline appears only after the tap, on the element that was already chosen.
- With no parent/child stepping, a tap on a phone selects whatever leaf element is under the finger, with no way to widen it.
- Text-range annotation depends on `mouseup` with a live selection. A long-press selection on iOS or Android does not reliably end with a `mouseup`, so text ranges are probably hard or impossible to annotate on touch. I found no test covering it: no file under `test/` uses `hasTouch`, `isMobile` or `tap(`.

## 5. Size of the injected SDK and its dependencies

**Observed.**

- The SDK is not a built file. `GET /sdk.js` assembles it per request by calling `Function.prototype.toString()` on `createArtifactSdk` and about 20 helper functions and wrapping them in an IIFE (`src/server.js:3176-3222`). It is not minified. The artifact gets one tag: `<script src="/sdk.js?key=...">` (`src/html-transform.js:29`).
- It has no runtime dependencies in the browser. The only imports are three sibling modules that are serialized in (`src/artifact-sdk.js:3-5`).
- Source sizes: `artifact-sdk.js` 119,283 bytes; with `mermaid-node.js`, `table-cell.js` and `artifact-revisions.js`, 142,185 bytes. Those four files gzip to 41,084 bytes (`gzip -9`).
- The outer page script, `chrome-client.js`, is 204,992 bytes, 56,037 gzipped.
- The npm package is 15.5 MB unpacked in 45 files (`npm view lavish-axi dist`), mostly the vendored Excalidraw whiteboard bundle and fonts (`scripts/build.js:60-83`).
- Node dependencies: `express`, `ws`, `chokidar`, `parse5`, `open`, `cross-spawn`, `axi-sdk-js`, `daisyui`, `@tailwindcss/browser` (`package.json`). Node 22 or later.

**Inference.**

- The served SDK is roughly 120-140 KB raw and about 40 KB gzipped. The 41 KB figure is the gzip of the source files, which include some code that is not serialized and comments that are; the true served size was not measured.
- Most of the SDK is not annotation. By line ranges, the layout audit is about 1,000 lines (1337-2120), attachments about 500, Mermaid about 250. The pick, highlight, selector and card code is roughly 500 lines. A hover-note equivalent of that subset fits in a few KB gzipped.

## 6. What it deliberately does not do

| Topic | Finding | Source |
| --- | --- | --- |
| Persistence across sessions | Delivered prompts are deleted from `state.json` on poll. A flat chat transcript is kept, capped at 5 MiB. State is per machine, outside the repo, never in git. Unsent notes live in one tab's `sessionStorage`. | `src/session-store.js:620-648`, `src/chat-messages.js:18`, `README.md:210`, `:261` |
| Threads | None per annotation. There is one linear conversation per session; an agent reply is a session-level chat entry, not a reply to a note. | `src/session-store.js:722`, `README.md:207` |
| Resolve state | None for human notes. The delivery receipts (Seen, Working, Done) are per batch, and a reply "marks every note the agent has seen as Done". A real lifecycle (open, queued, resolved, returned) exists only for machine-detected layout issues, and is cleared by re-checking, "never by the user being asked to declare it settled". | `README.md:179-183`, `:219`, `VISION.md` "Nothing interrupts the human" |
| Versioning | None of notes. Optional "revisions legend" is the agent declaring what it changed, inside the artifact. | `README.md:184-187` |
| Multiple people | Excluded: "it is not a place for two people to collaborate with each other". One tab owns a review at a time. | `VISION.md` "Scope", `README.md:172` |
| Pages that are not standalone HTML files | Excluded: "Lavish reviews HTML, and it does not adopt Markdown, PDFs, or images". Sessions are keyed by a file path on disk. The artifact is served by Lavish's own server in an opaque-origin sandbox, so a running app with its own dev server, routes, cookies or root-relative asset paths is out of scope; root-prefixed paths "will not resolve". | `VISION.md` "Scope", `README.md:161`, `:189` |
| MCP | Excluded: "It is not an MCP server". | `VISION.md` "Scope" |
| Agent writing notes | The agent can post chat replies only. It cannot create an annotation on an element. | `README.md:277`, `:310` |

Also relevant: the npm build sends anonymous usage telemetry by default, opt-out with `LAVISH_AXI_TELEMETRY=0` (`README.md:321-330`).

## 7. AXI design principles relevant to an agent-facing CLI

The ten principles, from `axi/README.md:79-92` (generated from `principles.yaml`):

1. Token-efficient output: TOON instead of JSON.
2. Minimal default schemas: 3-4 fields per list item.
3. Content truncation, with size hints and a `--full` escape hatch.
4. Pre-computed aggregates: counts and statuses that save a round trip.
5. Definitive empty states: an explicit "0 results".
6. Structured errors and exit codes; idempotent mutations; no interactive prompts; fail loudly on unknown flags.
7. Ambient context: opt-in session hooks first, then an on-demand skill.
8. Content first: no arguments shows live data, not help.
9. Contextual disclosure: a next-step suggestion after each output.
10. A consistent way to get help per subcommand.

How lavish-axi applies them (Observed): bare `lavish-axi` lists live sessions (`README.md:273`); every poll result carries `next_step` (`src/cli.js:611`); errors are typed, e.g. `NOT_FOUND`, `LISTENER_ACTIVE`, with suggested commands (`src/cli.js:594-597`); the installable skill is a stub that points at `--help` so it cannot go stale (`README.md:56`); identity is the file path, so the agent holds no opaque IDs (`README.md:161`).

The benchmark numbers in `axi/README.md` are the author's own and were not checked.

## Implications for hover-note

**Reuse or copy (MIT; keep the copyright notice if code is copied rather than rewritten).**

- The event wiring: capture-phase `mouseover`/`mouseout`/`mouseup`/`click` on `document`, with `preventDefault` + `stopPropagation` on click, and early return for the library's own UI. About 90 lines (`src/artifact-sdk.js:2668-2756`).
- The skip list for native controls and ARIA widgets (`isNativeInteractiveControl`, 117-130), and Cmd/Ctrl+click to follow a link while targeting.
- The selector builder with a uniqueness check and distrust of repeated ids (802-832, 876-882). Good as one field of an Anchor, not as the whole Anchor.
- The text-range target shape (1185-1214), extended with prefix and suffix context.
- One shadow root with `:host{all:initial}` for all library UI, and stopping key events at it (2287-2304).
- Text highlight from `range.getClientRects()` (1247-1260).
- IME-safe Enter handling in the card (`isComposing`, `keyCode 229`, 2507).
- CLI shape: content-first bare command, typed errors, a next-step line, stdout reserved for the result.
- The receipt idea (Seen, Working, Done) as states derived from what the Backend observes, with no extra work for the agent.

**Avoid.**

- Inline `outline` on the target element. Draw an overlay box in the shadow root instead, so the host page is never mutated.
- A positional-only Anchor. `tag:nth-of-type` chains break on the first structural edit; hover-note Annotations must survive that. Record several independent signals (id or test-id, role and accessible name, text quote with context, structural path) and score candidates.
- A per-load `uid` plus a full DOM snapshot as the way to give the agent context. It costs tokens on every delivery and is meaningless after a reload.
- Destructive delivery. hover-note Annotations are files in git; reading must not delete.
- Assembling the browser script with `fn.toString()` at request time. It prevents minification and tree-shaking, and the result here is about four times the whole hover-note size budget.
- Resending a long instruction paragraph with every result. Keep the next-step hint to one line.
- Depending on a local server and a sandboxed iframe for the default path. hover-note runs inside the page it annotates.
- Telemetry on by default.

**Open questions.**

1. Touch Targeting is unsolved here, so there is nothing to copy. What is hover-note's inspect-mode equivalent: tap to pick then step to parent, or press-and-drag with a live outline? How are text ranges picked when there is no `mouseup`?
2. Should hover-note offer parent/child stepping on desktop as well (lavish-axi does without it)?
3. Shadow DOM and same-origin iframes: use `composedPath()[0]` and record a path through shadow hosts, or declare both out of scope for the core?
4. Does the default file Backend need any long-poll equivalent, or is "the agent reads the files" enough? If a wait command is wanted, it implies a running process, which the file Backend otherwise avoids.
5. Is TOON worth it for hover-note's agent output? The encoder is a dependency of the CLI, not of the browser core, so the size budget is not affected, but the files in git still need a format humans can diff.
6. Should the Anchor include any rendered evidence (bounding box, viewport, optional screenshot)? lavish-axi records none and relies on reviewer-attached images.
7. Table-cell row and column labels are a cheap, high-value addition to an Anchor. Core or plugin?

**Not verified.** Nothing was run. The served SDK size, the exact TOON rendering, and all touch behaviour are read from source or inferred, not measured.
