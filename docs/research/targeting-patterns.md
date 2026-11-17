# Targeting patterns: how existing tools pick elements on desktop and on touch

Research note, 2026-11-17. Ticket: "How do existing tools do fast element picking on desktop and on touch?"

**Method.** Source code was read, never installed or run. Nothing here was confirmed in a browser. Web retrieval used the Parallel, Exa and Tavily search tools (one Parallel query was rate-limited and redone through Exa and Tavily).

Claims are marked:

- **Observed**: read from the cited file and lines, or from the cited page.
- **Inference**: my reading of what the code or spec implies.
- **Synthesis**: my recommendation. Not a sourced fact.

**Sources read.**

| Tool | Version read | Where the picker lives |
| --- | --- | --- |
| React Grab | `aidenybai/react-grab` @ `ea4bbec` (2026-08-22), v0.2.0 | `packages/react-grab/src/` (paths below are relative to this) |
| Agentation | `benjitaylor/agentation` @ `0e3236e` (2026-09-21) | `package/src/` |
| SitePing widget | `@siteping/widget` 0.12.5 from npm; original sources recovered from `dist/index.js.map` | `src/annotator.ts`, `src/dom/anchor.ts` |
| lavish-axi SDK | `kunchenguid/lavish-axi` @ `d0c1667` (2026-11-16) | `src/artifact-sdk.js` |
| Eruda | `liriliri/eruda` @ `0c55928` (2025-08-01); its picker is in `liriliri/chobitsu` @ `819b9d0` (2025-08-17) | `chobitsu/src/domains/Overlay.ts` |
| uBlock Origin | `gorhill/uBlock` @ `a34ea75` (2026-11-17) | `src/js/scriptlets/epicker.js`, `src/js/epicker-ui.js`, `platform/mv3/extension/js/` |
| Chrome DevTools overlay | `chromium/chromium` `main`, fetched 2026-11-17 | `third_party/blink/renderer/core/inspector/inspect_tools.cc`, `inspector_overlay_agent.cc` |
| vConsole | `Tencent/vConsole` @ `de7026d` (2026-03-27) | no picker found (see section 3) |

## Summary

1. Every tool that works reliably finds the element with `document.elementFromPoint(x, y)`, then repeats the call inside each open shadow root and same-origin iframe. Nobody trusts `event.target` alone for the final answer.
2. Two families exist for blocking the page while picking. **Capture listeners** on `window`/`document` swallow `click` (lavish, Agentation, chobitsu). A **full-viewport shield** takes all input, and is switched to `pointer-events: none` for the instant of each hit test (SitePing, uBlock, React Grab). The shield is the only family that also stops `:hover` styles, focus and touch gestures.
3. The highlight is a single box positioned from `getBoundingClientRect()` in every tool except lavish, which writes `outline` onto the page's own element. No surveyed tool positions the highlight with CSS anchor positioning. Only Agentation uses the top layer (`popover="manual"`).
4. On touch, the working precedents are **tap to pick** (SitePing, uBlock, Chrome remote inspect), **tap to highlight, tap again to confirm** (uBlock zapper), **drag to scrub with live highlight** (Eruda/chobitsu), and **long-press** (Figma's native app). I found no web tool that uses a loupe.
5. Touch gestures are suppressed with `touch-action: none` on a shield, set before the gesture starts. The Pointer Events spec says panning cannot be stopped by cancelling a pointer event.
6. Measured costs are scarce. The only numbers are self-reported profiling notes in React Grab's source and one Chrome figure. See section 4.

## 1. Desktop: how each tool does it

### 1.1 Comparison

| | Find element | Move event | Block the page | Highlight | Parent/child stepping | Text ranges |
| --- | --- | --- | --- | --- | --- | --- |
| lavish-axi | `event.target` | `mouseover`/`mouseout`, capture | capture `click` + `preventDefault` + `stopPropagation` | inline `outline` on the page element | none | `mouseup` + `getSelection()` |
| Agentation | `elementFromPoint`, deep | `mousemove`, not throttled | capture `click`; optional | one `position: fixed` div | Cmd/Ctrl "pierce" picks a smaller element; no stepping | selection read at click time |
| chobitsu (Eruda) | `elementFromPoint`, not deep | `mousemove`, capture, not throttled | capture `click` + `stopImmediatePropagation` | `luna-dom-highlighter` in a shadow root | none in the picker (DOM tree afterwards) | none |
| SitePing | `elementFromPoint`, deep (shadow only) | no hover highlight at all | full-viewport overlay, `pointer-events: auto` | drawn rectangle only | none | none in the annotator |
| uBlock Origin | `elementFromPoint`, not deep | `mousemove`, passive, one per rAF | full-viewport iframe in a closed shadow root | SVG path in the iframe | depth/specificity slider after the pick | none |
| React Grab | `elementFromPoint` then `elementsFromPoint` fallback, deep | `pointermove`, capture, passive, 32 ms throttle | per-document shield + capture listeners | `<canvas>` in an open shadow root | arrow keys | none (text bounds used only for hit testing) |
| Chrome DevTools | engine hit test | native mouse move | engine swallows input | separate overlay page painted by the engine | Elements tree | none |

### 1.2 Finding the element

**`event.target` (Observed).** lavish uses the raw target of `mouseover` and `click` (`src/artifact-sdk.js:2668-2685`, `2718-2754`). This is the smallest possible picker, and it works for lavish because its document is a sandboxed iframe it owns.

**Deep `elementFromPoint` (Observed).** The common loop: call `document.elementFromPoint`, then while the result has an open `shadowRoot`, call `shadowRoot.elementFromPoint` with the same coordinates.

- SitePing: `src/dom/anchor.ts:164-176`. It stops if the inner result's root is not that shadow root.
- Agentation: `package/src/utils/hit-testing.ts:10-26`. Also descends into same-origin iframes, converting coordinates by the frame's offset and scale.
- React Grab: `utils/get-deep-element-at-point.ts:5-33`. Same, with `convertParentPositionToIframe`.

**`elementsFromPoint` as fallback (Observed).** React Grab takes the top element first and only walks the full z-ordered stack when the top element is not pickable, for example a transparent overlay (`utils/get-element-at-position.ts:191-205`, `utils/get-deep-fallback-element-at-point.ts`). Agentation uses the stack for its "pierce" modifier and prefers the first non-container element or one with direct text, then the smallest by area (`hit-testing.ts:94-118`).

**Filtering full-page overlays (Observed).** React Grab rejects elements that cover at least 90% of the viewport and are `position: fixed|absolute` with a transparent background, opacity under 0.1, or a z-index over 1000 (`utils/is-valid-grabbable-element.ts:34-44`, constants in `constants.ts:226-228`). It calls `getComputedStyle` only for elements that pass the size check, and comments that `getComputedStyle` "is the expensive call in this function" (`is-valid-grabbable-element.ts:95-97`).

**`composedPath()` (Observed).** Agentation uses `e.composedPath()[0] || e.target` only to ask "did this event start inside my own UI?" (`page-toolbar-css/index.tsx:1836`, `1947`). It still hit-tests by point for the result.

**Closed shadow roots (Inference).** No tool reaches into closed shadow roots. `element.shadowRoot` is `null` for a closed root, so every loop above stops at the host. For hover-note that means a closed-root component is one Targeting unit. Confidence: high (DOM behaviour), not re-verified this session.

**Engine hit test (Observed).** Chrome's inspect tool runs a Blink hit test with `kMove | kReadOnly | kAllowChildFrameContent`. Holding Shift adds `kIgnorePointerEventsNone`, so elements with `pointer-events: none` become pickable. Text nodes resolve to their parent. User-agent shadow content resolves to its host (`inspect_tools.cc:65-82`, `212-225`). A page script cannot do the `pointer-events: none` trick without `elementsFromPoint`-style heuristics.

### 1.3 Events

**`mousemove` vs `pointermove` (Observed).** Only React Grab uses pointer events. lavish, Agentation, chobitsu, uBlock and SitePing use mouse events, with separate touch listeners where they support touch.

**Capture phase (Observed).** React Grab attaches every pointer listener to `window` with `capture: true`. Its comment gives the reason: "so detection survives apps that stopPropagation() pointermove below window level (gesture libraries, analytics/session-replay SDKs)" (`core/index.tsx:3051-3055`). chobitsu attaches to `document.documentElement` with capture (`Overlay.ts:189-200`).

**Throttling (Observed).**

- uBlock stores the last coordinates and schedules one `requestAnimationFrame` per frame (`platform/mv3/extension/js/tool-overlay-ui.js:152-167`).
- React Grab throttles hit tests to 32 ms with `setTimeout` (`constants.ts:77`, `core/index.tsx:1988-2003`) and reuses the last result when the pointer moved 2 px or less within 16 ms (`constants.ts:286-287`, `get-element-at-position.ts:156-166`).
- Agentation and chobitsu run a hit test on every event.

**Is manual rAF throttling needed? (Observed + Inference).** Chrome has dispatched `mousemove`, `pointermove`, `touchmove` and `wheel` once per frame, just before rAF callbacks, since Chrome 60 (https://developer.chrome.com/blog/aligning-input-events). The Pointer Events spec says user agents "MAY delay dispatch of the pointermove event" (https://www.w3.org/TR/pointerevents4/). A 2019 survey found Safari did not align `mousemove` (https://nolanlawson.com/2019/08/14/browsers-input-events-and-frame-throttling); I did not find a current figure for Safari. Inference: a one-line rAF guard is still worth its bytes.

### 1.4 Blocking the page while picking

**Capture listeners only (Observed).**

- lavish: capture `click` calls `preventDefault()` and `stopPropagation()` (`artifact-sdk.js:2737-2738`). Cmd/Ctrl-click on a link is let through (`2729-2736`). Native controls are exempt so they still work (`isInteractiveControl`, `1228-1230`).
- Agentation: capture `click`; blocking is a setting. Its comment explains why `stopPropagation` is needed for every target: framework handlers "are delegated to the root and would otherwise still fire" (`index.tsx:2003-2012`).
- chobitsu: capture `click` with `preventDefault()` and `stopImmediatePropagation()` (`Overlay.ts:174-187`).

Inference: capture listeners stop `click`, but not `mousedown`/`pointerdown` handlers, `:hover` styles, focus changes or drag starts, unless each is also intercepted. React Grab intercepts `pointerdown`, `pointerup`, `click` and `contextmenu` separately (`core/index.tsx:3058-3214`).

**Shield (Observed).**

- SitePing: one `position: fixed; inset: 0` div with `pointer-events: auto`, appended to `document.body` (`annotator.ts:242-262`, `365`). Before each hit test it sets `pointer-events: none` on the shield, calls `deepElementFromPoint`, then restores it (`annotator.ts:649-652`). It also sets `document.body.style.overflow = "hidden"`, so the page cannot scroll while annotating (`annotator.ts:229-231`).
- uBlock: a full-viewport iframe inside a closed shadow root on `document.documentElement` (`tool-overlay.js:317-325`). The iframe gets a `[clicked] { pointer-events: none !important }` rule that is toggled around `document.elementFromPoint` (`tool-overlay.js:73`, `277-294`).
- React Grab: a shield per document, with `contain: strict`, at z-index 2147483644 (`utils/create-hit-test-shield.ts:83-97`). Same-origin iframes are cut out of it so they still scroll. Wheel events on the shield are re-applied to the real scroll container underneath (`create-hit-test-shield.ts:117-144`).

**Why React Grab moved to a shield (Observed, self-reported).** Its earlier approach set `pointer-events: none` on the root. The source comment says that because `pointer-events` is inherited, flipping it on the root "restyles every element in the document (profiled at 4.2ms on 5.6k elements and 31-35ms on 16k)" (`create-hit-test-shield.ts:13-19`). It also avoids toggling `display` on the shield, which rebuilt a composited layer on every hit test (`create-hit-test-shield.ts:21-25`).

**The page disabling its own content (Observed).** Radix-style modals set `body { pointer-events: none }`. React Grab counters with `html, body { pointer-events: auto !important; }` while active (`utils/pointer-events-freeze.ts:6-24`).

### 1.5 Drawing the highlight

- **Inline outline (Observed).** lavish sets `el.style.outline` and `outlineOffset` on the hovered element and clears them with `el.style.outline = ""` (`artifact-sdk.js:1232-1240`). Inference: this needs no measurement and follows scroll and transforms for free, but it mutates the page, erases any inline outline the page had, and is clipped by `overflow: hidden` ancestors.
- **One fixed box (Observed).** Agentation renders one `position: fixed` div from the element's viewport rect (`index.tsx:4407-4417`, `styles.module.scss:899-908`). SitePing's rectangle is also `position: fixed; pointer-events: none` (`annotator.ts:594`).
- **SVG in an iframe (Observed).** uBlock builds an SVG path: an "ocean" rectangle for the viewport plus "islands" for each highlighted rect, skipping off-screen rects (`tool-overlay.js:228-247`). It re-measures on `scroll` and `resize` with passive listeners (`tool-overlay.js:75-76`). For zero-size elements it unions the children's rects (`tool-overlay.js:194-226`).
- **Canvas (Observed).** React Grab draws on a viewport-sized `<canvas>` in an open shadow root (`components/overlay-canvas.tsx:87-106`, `utils/mount-root.ts:88`), caches rects for 16 ms and border radius for 200 ms (`utils/create-element-bounds.ts:42-78`, `constants.ts:83-89`).
- **Top layer (Observed).** Agentation is the only tool that uses the Popover API. When mounted inside a modal it sets `popover="manual"` on a zero-size host and calls `showPopover()`. Its comment: "Keep DOM ownership inside the modal for focus/aria, while escaping its transforms and overflow so all coordinates remain viewport-relative" (`page-toolbar-css/use-feedback-portal.ts:31-38`).
- **CSS anchor positioning (Observed: none use it).** A search for `anchor-name`, `position-anchor` and `anchorName` found nothing in any of the six tools. It became Baseline Newly Available with Firefox 147 on 2026-01-13; Chrome 125 and Safari 26 had it earlier (secondary sources, e.g. https://www.testmuai.com/learning-hub/css-anchor-positioning-browser-support; MDN's module page did not show a Baseline banner in the excerpt I fetched). Inference: using it would mean writing `anchor-name` onto the page's element, which is the same page mutation as lavish's outline.

**Scroll, transforms, fixed and sticky (Observed + Inference).** Every box-drawing tool reads `getBoundingClientRect()`, which returns viewport coordinates after transforms, so a `position: fixed` box lines up with fixed, sticky and transformed targets without special cases. What it cannot show is rotation: a rotated element gets its axis-aligned bounding box. All tools re-measure on scroll. React Grab uses a capture-phase passive `scroll` listener on `window`, which also catches scrolls of inner containers, plus `visualViewport` `resize` and `scroll` (`core/index.tsx:3319-3360`). Agentation does the same for scroll (`index.tsx:1367`).

**A fixed box inside a transformed ancestor (Inference).** `position: fixed` is relative to the nearest ancestor with a transform, filter or containment, not the viewport. A tool that mounts its box inside such an ancestor draws in the wrong place. Mounting on `document.documentElement` (uBlock, chobitsu) or in the top layer (Agentation) avoids it. Confidence: high, standard CSS behaviour, not re-verified this session.

### 1.6 Excluding the tool's own UI

- Attribute markers checked with a shadow-crossing `closest`: Agentation (`hit-testing.ts:4`, `65`), SitePing (`data-siteping-ignore`, `annotator.ts:262`), React Grab (`data-react-grab-ignore-events`, `core/index.tsx:3027`).
- A wrapper element found in `composedPath()`: Agentation (`index.tsx:513`).
- Stopping events at the tool's shadow root so they never reach page listeners: lavish (`artifact-sdk.js:2298`).
- A closed shadow root plus a random attribute name: uBlock (`tool-overlay.js:321-325`).

### 1.7 Stepping to parent or child

- **Arrow keys (Observed).** React Grab: Up moves to the next element in the `elementsFromPoint` stack at the element's centre, Down goes back along the history, Left/Right move between DOM siblings (`core/arrow-navigation.ts:24-80`).
- **Slider after the pick (Observed).** uBlock walks `parentElement` from the picked element to the root, builds candidate selectors, and shows a slider to move between broader and narrower matches (`platform/mv3/extension/js/scripting/picker.js:81-225`, `picker-ui.js:32-33`, `180-183`).
- **Modifier key (Observed).** Agentation re-evaluates the same point when Cmd/Ctrl goes down or up (`index.tsx:1846-1850`). Chrome uses Shift to include `pointer-events: none` elements (`inspect_tools.cc:216-217`).
- **Scroll wheel (Observed: none).** No surveyed tool steps with the wheel. React Grab forwards wheel to page scrolling instead.

### 1.8 Text ranges next to element picking

- lavish: capture `mouseup` reads `document.getSelection()`. If there is a usable selection it opens the card and sets `ignoreNextClick` so the click that follows the drag is swallowed (`artifact-sdk.js:2698-2715`, `2739-2742`).
- Agentation: reads the selection at click time and stores up to 500 characters with the element (`index.tsx:2049-2053`). Its cursor rule gives text elements a text cursor and everything else a crosshair (`index.tsx:1795-1798`).
- Vercel's toolbar documents the same split: "click on the page or highlight text to place your comment" (https://vercel.com/docs/comments).
- React Grab, SitePing's annotator, chobitsu and uBlock have no text-range path. React Grab sets `user-select: none` on the body when a drag starts and restores it after (`core/index.tsx:2058`, `1738`).

Inference: a shield and native text selection conflict. A shield takes the mouse-down that would start a selection. lavish and Agentation can offer text ranges because they do not use a shield.

Platform pieces for a shield-compatible text path: `document.caretPositionFromPoint()` is Baseline Newly Available since December 2025 (https://developer.mozilla.org/en-US/docs/Web/API/Document/caretPositionFromPoint). The CSS Custom Highlight API styles a `Range` "without affecting the DOM structure" and is Baseline Newly Available since June 2025 (https://developer.mozilla.org/en-US/docs/Web/API/CSS_Custom_Highlight_API).

### 1.9 The "no DOM work until annotate mode is on" constraint

- chobitsu fails it: its listeners are attached at module load and return early when `inspectMode === 'none'` (`Overlay.ts:143-144`, `189-200`).
- lavish fails it the same way: listeners are attached at load and check an `annotationMode` flag (`artifact-sdk.js:2668-2676`).
- uBlock passes: `start()` adds listeners and CSS, `stop()` removes them (`tool-overlay.js:40-100`).
- SitePing passes: the overlay and its listeners are created in `activate()` (`annotator.ts:203-365`).

## 2. Touch: precedents

| Precedent | Gesture | How the page is kept from reacting | Source |
| --- | --- | --- | --- |
| SitePing | tap picks the element; drag draws a rectangle | shield with `touch-action: none`, `-webkit-touch-callout: none`, `user-select: none`; non-passive `touchstart`/`touchmove` call `preventDefault()`; body `overflow: hidden` | `annotator.ts:243`, `349-351`, `563-566`, `612-615` |
| uBlock picker | tap picks; then slider refines | iframe shield; tap = under 32 px and under 200 ms, then `preventDefault()` on `touchend` | `epicker-ui.js:382-413`, `zapper-ui.js:44-74` |
| uBlock zapper | tap highlights; tap the highlight again to act | same | `epicker.js:969-978`, `zapper-ui.js:27-40` |
| uBlock classic picker | swipe right/left hides/shows the dialog | same; swipe = over 64 px within ±22.5° of horizontal | `epicker-ui.js:414-432` |
| Eruda / chobitsu | finger down and drag scrubs a live highlight; lifting picks | capture `touchend` calls `preventDefault()` + `stopImmediatePropagation()`; nothing stops scrolling | `Overlay.ts:132-139`, `174-200` |
| Chrome remote inspect | tap picks; mode turns off after one tap | engine consumes `GestureTap` and `pointerdown` | `inspect_tools.cc:284-302`; https://developer.chrome.com/docs/devtools/remote-debugging |
| React Grab | touch pointer sets "touch mode"; body gets `touch-action: none` while active; `setPointerCapture` on `pointerdown` | body `touch-action: none`; `preventDefault()` + `stopImmediatePropagation()` on `pointerdown`/`pointerup`/`click` | `core/index.tsx:3025-3030`, `353`, `3096-3109` |
| Figma mobile app | long-press a canvas location to add a comment | native app | https://help.figma.com/hc/en-us/articles/1500007537281-Guide-to-the-Figma-mobile-app |
| Agentation | none | README: "Desktop browser (mobile not supported)" | `README.md:72` |
| lavish-axi | none | no touch listeners in the annotation path | `artifact-sdk.js` |
| vConsole | none | no picker: no `elementFromPoint` or highlight code under `src/`; its Element panel is a tree | repo grep |

Details worth keeping:

- **SitePing treats a small gesture as a tap.** Under 10 px on either axis, the gesture "points at the element under it"; the comment gives the reason: "a fingertip hides the very corner it is trying to place" (`annotator.ts:15-16`, `676-686`). On touch its instruction text changes to a tap instruction, chosen by `isCoarsePointer()` (`annotator.ts:207`, `316`).
- **SitePing clamps a tapped element's box to the viewport**, because "a tapped element can be far taller than the screen" (`annotator.ts:516-520`, `552`).
- **uBlock's two-tap confirm** exists because there is no hover preview on touch: "first highlight the element and remove only when tapping again the highlighted area" (`epicker.js:969-970`).
- **chobitsu's scrub** needs `elementFromPoint` on every `touchmove` because touch events keep the target of the first touch. Eruda turns off the size tooltip on mobile (`eruda/src/Elements/Elements.js:236-241`).
- **Chrome's remote inspect** is one-shot: "Select Element is disabled after the first touch, so you need to re-enable it every time" (Chrome DevTools docs, URL above).

### 2.1 Platform rules that decide the design

**Panning cannot be cancelled from a pointer event (Observed).** Pointer Events spec: panning and zooming "cannot be suppressed by canceling a pointer event. Authors must instead use touch-action" (https://www.w3.org/TR/pointerevents/). When the browser starts to pan, the pointer stream ends with `pointercancel` (https://developer.mozilla.org/en-US/docs/Web/API/Element/pointercancel_event).

**`touch-action` must be in place before the finger lands (Observed).** MDN: "After a gesture starts, changes to touch-action will not have any impact on the behavior of the current gesture" (https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/touch-action). Inference: long-press-then-scrub cannot switch off scrolling at the moment the long-press fires. `touch-action: none` has to be set when annotate mode starts, which is what SitePing and React Grab do.

**Touch listeners on `window`, `document` and `body` are passive by default in Chrome (Observed).** Since Chrome 56, `preventDefault()` in such a `touchstart` or `touchmove` listener is ignored unless `{ passive: false }` is passed (https://developer.chrome.com/blog/scrolling-intervention). SitePing passes `{ passive: false }` on its overlay element (`annotator.ts:349-350`). The same page says: "To suppress the default behavior of a tap (such as the generation of a click event), call preventDefault() inside of a touchend listener." That is what chobitsu and uBlock do.

**Touch pointers are captured by the element first touched (Observed).** Spec, "Implicit pointer capture": touch input "SHOULD behave exactly as if setPointerCapture was called on the target element just before the invocation of any pointerdown listeners" (https://www.w3.org/TR/pointerevents/). Inference: during a scrub, `event.target` never changes, so the element under the finger must come from `elementFromPoint(clientX, clientY)`. With a shield the target is the shield anyway.

**Long-press callouts and context menus (Observed).** `-webkit-touch-callout: none` disables the iOS touch-and-hold callout; it is non-standard (https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/-webkit-touch-callout). WebKit bug 213953 (2020) reports that Safari on iOS does not fire `contextmenu` and that `user-select: none` disables the magnifier (https://bugs.webkit.org/show_bug.cgi?id=213953). A 2024 Pointer Events working-group thread repeats that iOS Safari "never fires a contextmenu event" while Android browsers do (https://lists.w3.org/Archives/Public/public-pointer-events/2024AprJun/0007.html). I did not verify the behaviour on iOS 26.

Inference: a shield covers all of these at once. The finger lands on the tool's own element, which has no text, no link and no image, so there is nothing for the browser to select, call out or open a menu for. Tools without a shield must set `user-select` and `-webkit-touch-callout` on the page and cancel `contextmenu`.

## 3. Not found or not verified

- **Framer's mobile comment mode.** No source found.
- **Figma in a mobile browser.** Only the native app's long-press is documented.
- **Safari Web Inspector picking by tapping the device.** Apple's "Inspecting iOS and iPadOS" page and WebKit's "Enabling Web Inspector" page describe connecting, not on-device element selection.
- **A loupe or magnifier in any web picking tool.** None of the tools read has one.
- **vConsole.** Absence of a picker is from a grep of `src/` at the commit above, not from documentation.
- **`luna-dom-highlighter` internals.** chobitsu delegates drawing to it; I did not read that package.
- **Independent benchmarks of `elementFromPoint` or `getBoundingClientRect`.** None found. See section 4.
- **Top-layer behaviour.** That a `popover` renders above every z-index and escapes ancestor transforms is from my knowledge of the HTML spec plus Agentation's comment. I did not fetch the spec text. The Popover API is Baseline Newly Available since January 2025 (https://developer.mozilla.org/en-US/docs/Web/API/Popover_API).
- **Safari's current input-event alignment** and **iOS 26 `contextmenu` behaviour**, as noted above.

## 4. Performance: what is measured or documented

| Claim | Source | Kind |
| --- | --- | --- |
| `document.elementFromPoint`, `getBoundingClientRect()` and `getClientRects()` force synchronous style and layout | Paul Irish, "What forces layout / reflow" (https://gist.github.com/paulirish/5d52fb081b3570c81e3a) | documented, no timings |
| `getComputedStyle()` typically forces style recalc and often layout | same | documented |
| Frame-aligned input cut hit tests by 35% in Chrome's experiment | https://developer.chrome.com/blog/aligning-input-events | vendor measurement |
| A hit test costs "~20ms on 100k-node DOMs" | React Grab `core/index.tsx:1991-1993` | source comment, method not shown |
| Flipping `pointer-events` on the root: 4.2 ms at 5.6k elements, 31-35 ms at 16k | React Grab `utils/create-hit-test-shield.ts:16-17` | source comment |
| Interleaving layout reads with style writes forced a second full recalc, "~59ms on a large app" | React Grab `utils/freeze-global-interactions.ts:34-37` | source comment |
| A universal `*` selector rule forced a recalc "profiled at ~62ms" | React Grab `constants.ts:203` | source comment |
| Toggling `display` on a viewport-sized fixed shield raised p95 frame time from 58 ms to 292 ms on a software rasterizer | React Grab `utils/create-hit-test-shield.ts:21-25` | source comment, CI only |
| Reading `visualViewport` on every move cost "~25ms of self time across a 3s" trace | React Grab `utils/get-visual-viewport.ts:13` | source comment |

**Inference.** The forced layout inside `elementFromPoint` is cheap when layout is already clean, which is the normal case for a page that is sitting still. The cost appears when the tool itself dirties style between reads. So the rules that matter are: one hit test per frame; all reads before all writes; never change an inherited property on the root; never use `*` rules; change only leaf elements the tool owns. Confidence: medium. It rests on React Grab's unreproduced notes plus the general layout model. hover-note should measure on its own test page.

## 5. Candidate patterns for desktop (Synthesis)

Size estimates are my guesses for minified + gzipped code. Nothing was built.

### D1. Capture listeners + deep `elementFromPoint` + one fixed box

On enter: add capture, passive `pointermove` on `window`; capture `click`, `pointerdown`, `keydown`; passive capture `scroll`. On move: store coordinates, one rAF, deep `elementFromPoint`, skip own UI, read the rect, move one `position: fixed; pointer-events: none` box. On leave: remove everything.

- **Why.** The smallest thing that is correct across shadow DOM, and it meets the no-DOM-work constraint by construction. The page stays scrollable and text stays selectable, so text ranges come almost free (`mouseup` + `getSelection()`, lavish's pattern). The box itself has `pointer-events: none`, so it never shows up in hit tests.
- **Size.** About 0.6-0.9 KB.
- **Pitfalls.** Page `:hover` styles, tooltips and menus still fire while picking. `mousedown` handlers and drag starts are not blocked unless intercepted one by one. Disabled form controls do not dispatch mouse events in some browsers, so the highlight will not follow the pointer over them (not verified this session). Pages with `body { pointer-events: none }` hide their content from the hit test (React Grab's Radix note).

### D2. Shield with a leaf-level gate

On enter: append one `position: fixed; inset: 0` shield. On each rAF: set the shield to `pointer-events: none`, hit-test, set it back. All input lands on the shield, so one set of listeners on one element replaces the capture set.

- **Why.** Blocks hover, focus, drag, click and navigation in one move, and is the same structure the touch patterns need (T1, T2). One code path for both input types.
- **Size.** About 0.9-1.4 KB including wheel forwarding.
- **Pitfalls.** Native text selection is gone; a text-range path must be built from `caretPositionFromPoint` and the Highlight API (roughly another 0.5 KB, and both are only Baseline since 2025). The wheel scrolls the page, not nested scrollers, unless deltas are forwarded (React Grab's `handleWheel`). Toggle `pointer-events`, never `display` (React Grab's software-rasterizer note). Same-origin iframes need their own shield or a hole.

### D3. Keyboard and stepping layer on top of D1 or D2

After a pick or while hovering: Up = `parentElement` (crossing shadow hosts), Down = back along the path that was climbed, Enter = confirm, Escape = cancel. Optional: Left/Right for siblings.

- **Why.** Every tool with refinement does it after the pointer pick, not instead of it (React Grab arrows, uBlock slider). Climbing by `parentElement` is far smaller than React Grab's stack-based version and is predictable.
- **Size.** About 0.3-0.5 KB.
- **Pitfalls.** Key handlers must run in capture and must not fire while the user is typing in the page or in a Comment box. Parents often share the child's rect, so a step can look like nothing happened; show the tag name or skip same-rect parents.

**Recommendation.** Prototype D1 and D2 side by side on one messy page (a modal, a carousel, a sticky header, a web component). The deciding question is whether hover-note needs text ranges on desktop more than it needs the page frozen. D3 goes on whichever wins.

## 6. Candidate patterns for touch (Synthesis)

All three assume D2's shield with `touch-action: none; -webkit-touch-callout: none; user-select: none`, created when annotate mode starts.

### T1. Tap, then refine

Tap picks the element under the finger and shows the highlight plus a small bar: Parent, Child, Confirm, Cancel. A second tap elsewhere re-picks.

- **Why.** The most common precedent (SitePing, uBlock, Chrome remote inspect), with uBlock's lesson that touch needs a visible confirm step because there is no hover preview. The Parent/Child buttons are D3 with buttons.
- **Size.** About 0.5-0.8 KB over D2.
- **Pitfalls.** With `touch-action: none` the page cannot scroll while picking. Offer a "scroll" toggle that drops the shield, or use `touch-action: pan-y` and accept that a vertical drag scrolls (then T2 is impossible). The bar must not sit under the finger or over the target. Clamp the highlight to the viewport for elements taller than the screen (SitePing). Use `pointerup`, not `click`, and call `preventDefault()` on `touchend`/`pointerup` so no synthetic click reaches the page.

### T2. Drag to scrub with live highlight, lift to pick

Finger down starts a live highlight that follows `elementFromPoint` under the finger; lifting commits. A label drawn above the finger names the element.

- **Why.** The closest thing to hover on touch, and Eruda shows it works. It lets the user correct a miss without lifting.
- **Size.** About 0.3-0.5 KB over D2, since it reuses the desktop move path with pointer events.
- **Pitfalls.** The finger hides the target, so the label must be offset upward and flipped near the top edge. Needs `touch-action: none` from the start, so no scrolling. Handle `pointercancel`. `event.target` is useless because of implicit capture; always hit-test by coordinates. Two-finger gestures should cancel, not pick.

### T3. Tap to preview, tap again to confirm

First tap highlights. Tapping the highlight confirms; tapping elsewhere moves it.

- **Why.** The smallest touch mode, taken from uBlock's zapper. No extra UI. A good fallback if T1's bar is too much chrome.
- **Size.** About 0.2-0.3 KB over D2.
- **Pitfalls.** No way to reach a parent that shares its child's box, or a child that fills its parent, without T1's buttons. Double-tap zoom must be off (`touch-action` handles it). Users may not discover the second tap without a hint.

**Not recommended for a first prototype.** Long-press as the entry gesture: it competes with the browser's own selection callout and context menu, `touch-action` cannot be changed after the gesture starts, and iOS Safari is reported not to fire `contextmenu`. A loupe: no web precedent found, and it needs a cloned or screenshotted view of the page, which will not fit the size budget.

**Detecting touch.** SitePing uses a coarse-pointer media query to choose instructions. React Grab switches per event on `pointerType === "touch"` (`core/index.tsx:3025`). Per-event is the better fit for laptops with touchscreens.

## 7. Accessibility: keyboard-only picking

**What exists (Observed).**

- SitePing is the only tool with a designed keyboard path. It records the focused page element before activating, moves focus to the overlay, and Enter annotates that recorded element. The overlay has `role="application"`, an `aria-label`, and `tabindex="0"`. Its comment explains why it is not `aria-hidden`: "focusing an aria-hidden element parks screen-reader users on a node that announces nothing" (`annotator.ts:210-227`, `254-262`, `376-384`, `492-512`). It cites WCAG 2.1.1.
- React Grab has arrow-key navigation once something is selected (`core/arrow-navigation.ts`).
- uBlock, chobitsu, lavish and Agentation handle Escape or a mode hotkey only.
- SitePing swaps to opaque surfaces under `prefers-reduced-transparency` or `prefers-contrast: more` and skips its entry animation under reduced motion (`annotator.ts:19-24`, `369`).

**Notes for hover-note (Synthesis).**

1. Start keyboard Targeting from `document.activeElement` (descend through `shadowRoot.activeElement`). If nothing is focused, start at the first element in the viewport.
2. Keys: Up = parent, Down = first child or back down the climbed path, Left/Right = siblings, Enter = confirm, Escape = leave annotate mode and restore focus to where it was.
3. Tab order only reaches focusable elements, so Tab alone cannot target a paragraph or image. Arrow-key tree walking is required, not optional.
4. Announce the current target through one `aria-live="polite"` region owned by the tool, with tag, accessible name or a text snippet, and depth. A highlight box alone tells a screen-reader user nothing.
5. Call `scrollIntoView({ block: "nearest" })` on the target when stepping, as uBlock does (`tool-overlay.js:157`).
6. Draw the highlight with enough contrast on any background (two-tone border) and do not animate it under `prefers-reduced-motion`.
7. If a shield is used, do not mark it `aria-hidden` while it holds focus (SitePing's note).
8. Add the live region and key handlers only when annotate mode turns on, to keep the no-DOM-work constraint.
