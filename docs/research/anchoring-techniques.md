# Which techniques keep an Anchor attached when the DOM changes?

Researched 2026-11-17. Scope: techniques for re-finding a DOM element or text range after the page changes, how each holds up, and a concrete multi-signal Anchor sketch that fits a ≤10KB gzipped, zero-dependency core.

## How to read this

- **[S]** = sourced fact, with the URL of the source that owns it. **[A]** = my analysis or synthesis. **[U]** = could not verify.
- Robustness ratings in the tables are **[A]** unless a cell cites a number.
- **Sizes were measured locally, not taken from Bundlephobia** (blocked in this environment). I downloaded each package's published tarball from the npm registry and gzipped the shipped file. No minifier was available, so where a package ships no minified build the figure is "comments and indentation stripped, then gzip -9", which is an upper bound on min+gzip. Each figure says which it is.
- **Maintenance status is from npm publish dates.** The GitHub API was not reachable, so I have no last-commit dates, issue counts or archive flags.
- Parallel Search hit its free-tier rate limit on the first call; web retrieval used Exa, Tavily, and direct reads of `raw.githubusercontent.com` and npm tarballs.

## Summary

1. **No single selector survives.** In the one controlled comparison I found (801 elements, 48 real sites, old vs. new version), every single-locator approach lost 39–83% of elements; voting across locators lost 27%; scoring every candidate element against many stored properties lost 11% [S: https://dl.acm.org/doi/full/10.1145/3571855, Table 5].
2. **The working pattern is the same in every serious tool:** store several independent signals, gather candidates from all of them, score each candidate against all signals, accept only above a threshold. Hypothesis does this for text [S: https://github.com/hypothesis/client/blob/main/src/annotator/anchoring/match-quote.ts], SitePing for elements [S: https://siteping.dev/docs/widget/anchoring], Healenium and Similo for test locators.
3. **An explicit, author-placed attribute beats everything else** and costs nothing at runtime. Playwright's docs call test ids "the most resilient way of testing" [S: https://github.com/microsoft/playwright/blob/main/docs/src/locators.md]. SitePing gives its `data-feedback-anchor` key the top prior [S: https://siteping.dev/docs/widget/anchoring].
4. **The real risk is a confident wrong match, not a lost one.** Similo "will always return a matching web element … unless a threshold is used" [S: https://dl.acm.org/doi/full/10.1145/3571855]; Hypothesis has an open report of a quote silently re-anchoring to a lookalike paragraph [S: https://github.com/hypothesis/client/issues/7571]. The "declare it lost" rule matters as much as the matching.
5. **It fits the budget.** The reusable parts are small: `@medv/finder` ≤2.0KB gz, `approx-string-match` ≤1.3KB gz (both upper bounds, measured). A purpose-built version of both plus a scorer should land around 3–4KB min+gz [A, estimate, not built].

## 1. What kinds of change are we defending against?

| Change | What breaks | What survives |
|---|---|---|
| Re-render / hot reload (same markup, new node objects) | Any cached node reference | Every stored signal; just re-resolve |
| Class-name churn (CSS-in-JS hashes, Tailwind edits) | Class-based CSS selectors | id, data-*, role, text, tag, position |
| Copy edits | Exact text match, accessible name | Selectors, structure, fuzzy text |
| Layout refactor (wrappers added/removed, element moved) | Absolute XPath, `nth-child` chains, child-combinator CSS | id, data-*, text, role + name |
| List reorder / insert | Sibling index, `nth-child` | Text, data-*, id |
| i18n / theme switch | All text signals | Selectors, structure, data-* |
| Element removed | Everything | Nothing: must be declared lost |

[A] The useful property of this table is that no row breaks every column. That is the whole argument for storing several signals.

## 2. W3C Web Annotation Data Model selectors

Source for this whole section: W3C Recommendation, 23 Feb 2017, https://www.w3.org/TR/annotation-model/ (§4.2). It is a data format, not an algorithm: it says nothing about how to re-find or when to give up.

| Selector | What it stores [S] | Robustness [A unless quoted] |
|---|---|---|
| `CssSelector` | `value`: a CSS path. Implementers "SHOULD use only commonly supported features of CSS". | As good as the selector generator. No fuzzy fallback. |
| `XPathSelector` | `value`: an XPath. | Same; absolute paths are the weakest locator measured (§5). |
| `TextQuoteSelector` | `exact` (required), `prefix`, `suffix` (each SHOULD be present). Text "MUST be normalized": tags removed, entities decoded. If several sequences match, the selection "SHOULD be treated as matching all of the matches". | Survives all structural change. Breaks on copy edits unless matched fuzzily. The spec defines no fuzzy matching. |
| `TextPositionSelector` | `start`, `end` character offsets in the normalized text stream. | The spec itself says it "is very brittle with regards to changes to the resource. Any edits or dynamically transcluded content may change the selection". Useful only as a hint. |
| `RangeSelector` | `startSelector`, `endSelector` (start inclusive, end exclusive), which SHOULD be the same class. | Inherits the weaker of its two selectors. |
| `refinedBy` | Chains a narrower selector onto a broader one. More than one `refinedBy` means alternatives that "will result in the same selection". | This is the right shape for a text range inside an element: element selector, refined by a text quote. |

Notes:

- [S] Offsets are in Unicode code points, not UTF-16 code units. JavaScript string indices are code units, so a strict implementation needs conversion. [A] hover-note need not be strict if it never exchanges annotations with other W3C tools.
- [S] `TextQuoteSelector` has a copyright note: quoting copies page text into the annotation. [A] For hover-note this is a privacy point too, because Anchors are committed to git (see open questions).
- [A] The model has no place for a confidence score, a fingerprint, or role + name. It is a subset of what a robust Anchor needs. Adopting the field names (`exact`, `prefix`, `suffix`, `start`, `end`) is free; adopting the JSON-LD envelope buys nothing unless interop is a goal.

A related browser-native format is the URL text fragment, `#:~:text=[prefix-,]textStart[,textEnd][,-suffix]`, which uses the same quote-plus-context idea [S: https://github.com/WICG/scroll-to-text-fragment]. [U] I did not check whether pages can resolve one to a `Range` programmatically; I do not think they can, so it is not a building block.

## 3. Text-range anchoring libraries

### Hypothesis client (current)

[S: https://github.com/hypothesis/client/blob/main/src/annotator/anchoring/html.ts, `types.ts`, `match-quote.ts`]

- **Stores** three selectors per annotation: `RangeSelector` (XPath to start/end container + offsets), `TextPositionSelector`, `TextQuoteSelector` with a fixed 32-character prefix and suffix (`contextLen = 32`).
- **Re-find order:** range → position → quote. Range and position results are accepted only if the resulting text equals the stored `exact` ("quote mismatch" otherwise). The quote step is the fuzzy one, and the stored position is passed to it as a `hint`.
- **Fuzzy quote match:** try `indexOf` first; only if there are no exact matches, run `approx-string-match` with `maxErrors = min(256, quote.length / 2)`. Each candidate is scored `(50·quote + 20·prefix + 20·suffix + 2·position) / 92`, each term a 0–1 similarity. The best score wins.
- **No acceptance threshold beyond `maxErrors`.** The code returns the top-scored match; there is no "best must beat second-best" check. The maintainers confirm the trade-off in the lookalike-paragraph issue: "there is a trade-off between precision and recall when anchoring annotations" [S: https://github.com/hypothesis/client/issues/7571].
- **Lost state:** annotations that fail to anchor are called orphans and listed in a separate tab instead of being hidden [S: https://web.hypothes.is/blog/showing-orphaned-annotations/].
- The 2013 design used four strategies and a modified `diff-match-patch` (Bitap) [S: https://web.hypothes.is/blog/fuzzy-anchoring/]. The current code has replaced that with `approx-string-match`.
- Licence BSD-2-Clause [S: package.json in the repo].

### `approx-string-match`

[S: https://github.com/robertknight/approx-string-match-js, npm tarball 2.0.3]

- Myers bit-parallel approximate search: finds a pattern in a text allowing up to k insertions, deletions or substitutions. Expected time O((k/32)·n). One function: `search(text, pattern, maxErrors)`.
- Size: 11.9KB raw / 4.3KB gz as published (heavily commented); **3.7KB / 1.3KB gz with comments stripped** (measured upper bound).
- MIT, zero dependencies. Latest publish 2026-11-15 (v2.0.3): maintained.
- [A] The best candidate if hover-note wants real fuzzy text matching: small, dependency-free, permissive, and the algorithm has no pattern-length limit.

### `dom-anchor-text-quote` + `dom-anchor-text-position`

[S: npm tarballs 4.0.2 and 5.0.0; https://github.com/tilgovi/dom-anchor-text-quote]

- Converts between a `Range` and a `TextQuoteSelector`, with optional prefix/suffix and a position `hint`.
- Depends on `diff-match-patch` (Apache-2.0, 78KB raw / 18KB gz) and works around that library's "hard 32-character pattern length limit" by slicing.
- Size with its dependency chain: **about 11.4KB gz comment-stripped** (measured upper bound). Over budget on its own.
- MIT. Latest publish 2017-02-10 (quote) and 2020-04-01 (position): unmaintained. Hypothesis itself no longer uses it.

### Apache Annotator

[S: https://incubator.apache.org/projects/annotator.html; npm `@apache-annotator/dom` 0.2.0]

- **Retired from the Apache Incubator on 2025-08-11.** Latest npm publish 2022-11-16.
- Implements `describe*`/`match*` for TextQuote, TextPosition, Range and CSS selectors. Its text-quote matcher is exact: the source says it "performs an exact search" and is essentially `indexOf(prefix + exact + suffix)` across text chunks. No fuzzy matching.
- Depends on `@babel/runtime-corejs3` and `optimal-select`. Apache-2.0.
- [A] Useful as a reference for chunked text-node traversal; not usable as a dependency.

## 4. CSS selector generators

These produce one selector string. None of them re-finds anything; they only decide how fragile that string is.

| Tool | How it builds the selector [S] | Size (measured) | Licence | Latest publish |
|---|---|---|---|---|
| `@medv/finder` 4.0.2 | Per level, candidates are id (penalty 0), class (1), attribute (2), tag (5), with `nth` as a last resort; searches for the lowest-penalty unique path, then shortens it. Default filters accept only "word-like" names: letters and hyphens, each word ≥3 chars, no run of 4+ consonants. Default attributes: `role`, `name`, `aria-label`, `rel`, `href`, word-like `data-*`. 1s timeout with a fallback path. | 8.5KB raw / 2.3KB gz as shipped (unminified ESM); ≤2.0KB gz comment-stripped. README claims 1.5KB min+gz. | MIT | 2024-12-13 |
| `css-selector-generator` 3.9.5 | Configurable selector types (id, class, tag, attribute, nth-child, nth-of-type), blacklist/whitelist, and an `ignoreGeneratedClassNames` option for Emotion / styled-components / MUI classes. Falls back to an `:nth-child` chain. | 11.6KB raw / **4.5KB gz** (shipped minified UMD) | MIT | 2026-11-15 |
| `optimal-select` 4.0.1 | Configurable `priority` (default id, class, href, src) and `ignore` filters; a separate `optimize` pass shortens the result. | 10.7KB raw / **3.7KB gz** (shipped minified) | MIT | 2017-01-07 (unmaintained) |
| `unique-selector` 0.5.0 | Tries selector types in a configured order (`ID`, `Class`, `Tag`, `NthChild`, optionally `Attributes`), with an `excludeRegex`. | ≤2.6KB gz comment-stripped (CJS, many small files) | MIT | 2020-10-13 (unmaintained) |
| Chrome DevTools "Copy selector" | Walks up from the node. In optimized mode it stops at the first ancestor with an `id`. Otherwise each step is the tag, plus class names only if they distinguish it from same-tag siblings, plus `:nth-child(n)` if classes are not enough. `input` with a `type` and no id/class gets `[type=…]`. No filtering of generated ids or classes. "Copy XPath" does the same with `//*[@id="…"]` and positional indices. | Not a library | BSD-style (Chromium) | Maintained |

Sources: finder source in the npm tarball and https://github.com/antonmedv/finder; https://github.com/fczbkk/css-selector-generator; https://github.com/Autarc/optimal-select; https://github.com/ericclemmons/unique-selector; https://github.com/ChromeDevTools/devtools-frontend/blob/main/front_end/panels/elements/DOMPath.ts.

[A] How they hold up:

- **Class churn:** finder is the only one whose defaults help. Its word-like filter rejects anything with a digit, so most hashes (`css-1x2y3z`, `sc-bdVaJa`) and most Tailwind utilities (`px-4`, `text-sm`, `ml-auto`) are skipped. Plain-word Tailwind classes (`flex`, `hidden`, `rounded`) still pass, and they are exactly the ones that get edited. SitePing adds its own class and id filters on top of finder for this reason (§7).
- **Layout refactor:** all of them fall back to `nth-child` when nothing unique exists, and that breaks on any sibling insert. DevTools output is the most positional.
- **Copy edits, i18n:** unaffected, since none use text.
- **Runtime cost:** generation runs `querySelectorAll` per candidate path, so cost grows with page size and with how non-unique the element is; finder caps it with a 1s timeout. It runs once, when the Annotation is created. Resolving is one `querySelector`.

[A] Verdict: a generated selector is a cheap, fast first candidate source and is worth storing because it is human- and agent-readable. It must never be the only signal, and it must be verified against other signals before it is trusted.

## 5. XPath, stable attributes, ARIA role + name

**XPath.** [S] In the Similo study, absolute XPath failed to locate 83% of elements across site versions, ID-relative XPath 59%, and ROBULA+ (the best XPath generator tested) 39% [https://dl.acm.org/doi/full/10.1145/3571855, Table 5]. ROBULA+ claims to reduce fragility "on average by 90% w.r.t. absolute locators" [S: https://github.com/cyluxx/robula-plus]. [A] XPath adds one thing CSS cannot do, which is match on text (`//button[text()="Save"]`), and loses one thing, which is that it cannot enter shadow roots [S: https://siteping.dev/docs/widget/anchoring]. `document.evaluate` is built into browsers, so resolving is free; a generator is roughly the size of a CSS one. I see no reason to store both a CSS path and an XPath: a plain tag + index path as data covers the same ground and can be compared fuzzily, which a string cannot.

**Stable `data-*` / test-id attributes.** [S] Playwright: "Testing by test ids is the most resilient way of testing as even if your text or role of the attribute changes, the test will still pass" [https://github.com/microsoft/playwright/blob/main/docs/src/locators.md]. Its generator scores the configured test id attribute 1 and `data-testid` / `data-test-id` / `data-test` 2, the best scores available (lower is better) [https://github.com/microsoft/playwright/blob/main/packages/injected/src/selectorGenerator.ts]. Agentation's defaults are `data-testid`, `data-test`, `data-qa`, `data-cy`, `data-component` [S: agentation package README, npm 3.1.2]. [A] Survives every change in §1 except removal of the attribute. Zero bytes, zero runtime cost. The only weakness is that someone has to have put it there, and that list items often share one value.

**ARIA role + accessible name.** [S] Playwright: "We recommend prioritizing role locators to locate elements, as it is the closest way to how users and assistive technology perceive the page", and "CSS and XPath are not recommended as the DOM can often change leading to non resilient tests" [locators.md, same URL]. In the generator, role + name scores 100, ahead of placeholder 120, label 140, alt 160, text 180, title 200, and far ahead of CSS id 500, tag 530, `nth` 10000 and the CSS fallback 10000000. [A] Survives class churn and layout refactors; breaks on copy edits and i18n because the name is usually the visible text. The cost is the problem: a correct accessible-name computation is large. `dom-accessibility-api`'s name-and-description module alone is 21KB raw / 5.5KB gz before its helpers (measured, npm tarball). [A] hover-note can afford only an approximation: explicit `role` or a small tag-to-role table, and name from `aria-label`, `aria-labelledby`, `<label for>`, `alt`, `title`, else trimmed text. [U] I did not check whether `Element.computedRole` / `computedName` are shipped in any browser by default.

## 6. Multi-signal scoring: test tools and the research

### Playwright

[S: selectorGenerator.ts, URL above] Codegen ranks candidates by the fixed scores listed in §5, penalises long text, and combines tokens of a chained selector by a position-weighted sum. It emits **one** best locator. It does not store alternatives and does not heal: if the locator stops matching, the test fails. Its CSS fallback walks up using id, then class combinations, then `nth-child`. Apache-2.0. [A] The ranking is the most carefully tuned public statement of "which signal to trust first" and is worth copying as an ordering. The generator itself depends on Playwright's own selector engines and role utilities, so it cannot be lifted.

### Chrome DevTools Recorder

[S: https://github.com/puppeteer/replay/blob/main/src/Schema.ts] Each recorded step stores `selectors: Selector[]`, "a list of alternative selectors that lead to selection of a single element", of kinds CSS, `aria/`, `xpath/` and `text/`, with the comment that recording all alternatives is "to improve reliability of the replay as some selectors might get outdated over time". Array-valued selectors step through shadow hosts. [A] This is a first-match cascade, the weakest way to use multiple signals (see SitePing's reasons in §7), but it confirms the storage shape.

### Healenium

- [S: https://healenium.io/docs/how_healenium_works] On a successful find it stores the element's path. When a locator later fails it "triggers the LSC algorithm", compares the stored path with the current page and "generates the list of healed locators", then uses the highest-scoring one.
- [S: https://github.com/healenium/healenium-web README] `score-cap` is the acceptance threshold, e.g. 0.5 means heal only if the match probability is ≥50%. `recovery-tries` bounds the candidates tried.
- [Secondary sources, not verified against Healenium's own code: https://nenyaglobal.atlassian.net/wiki/spaces/CTF/pages/8625192979/Cast-Healenium+Integration and https://home.wonlab.top/en/blog/llm-practice/llm-autotest/llm-autotest-series-10-self-healing-locators] The stored path is a list of nodes from `html` down, each `{tag, index, innerText, id, classes, other}`. Scoring is longest-common-subsequence path similarity plus fixed weights: tag 100, id 50, class 40, text 30, other attributes 30, with Levenshtein similarity on strings, normalised to 0–1. The stored path is updated after each successful find. A maintainer-side answer to "how does the algorithm work?" was to decompile the jar [S: https://github.com/healenium/healenium/issues/56], which is why I could not find a first-party description.
- Java, needs a backend service; not reusable. [U] Licence not checked.

### Similo and successors

[S: Nass, Alégroth, Feldt, Leotta, Ricca, "Similarity-based Web Element Localization for Robust Test Automation", ACM TOSEM 32(3), 2023, https://dl.acm.org/doi/full/10.1145/3571855; preprint https://arxiv.org/abs/2208.00677]

- **Method.** Store 14 properties of the target element. Later, compare each against the same property of every candidate element with a 0–1 comparison function, multiply by a weight, and sum. The highest total wins. Unlike voting across locators, a property helps even if it does not identify a unique element.
- **Properties:** tag, class, name, id, href, alt, absolute XPath, ID-relative XPath, is-button, location (x, y), area, shape (width/height), visible text, neighbour texts.
- **Weights:** only two values. 1.5 for tag, id, name, visible text and neighbour texts; 0.5 for the rest. The authors chose them by hand and say the result is therefore an underestimate.
- **Result:** on 801 elements across two versions of 48 popular sites, non-located elements were: absolute XPath 83%, ID-relative XPath 59%, Selenium IDE 51%, Montoto 47%, ROBULA+ 39%, multi-locator voting at its theoretical limit 27%, **Similo 11%**.
- **Cost:** 4 ms average to locate one element, growing with the number of candidates.
- **Stated weakness:** without a threshold it always returns something, including when the target has not loaded yet. Levenshtein on short visible text can prefer the wrong element ("Home & Garden" scored closer to "Home Improvement" than to "Home").
- **Follow-up** [S: Kluge & Stocco, Empirical Software Engineering 31(6), 2026, https://dl.acm.org/doi/10.1007/s10664-026-10903-6, abstract only]: a replication found that the successor VON Similo "tends to produce more false positives than Similo"; tuned parameters improved Similo by 5.62% on its original benchmark; a hybrid located 98.8% of elements with broken locators on a larger benchmark. [U] I read only the abstract, so I do not know which parameters changed.

[A] Similo is the strongest evidence in this document and it maps directly onto hover-note: it is cheap, needs no ML, and the five heavily weighted properties (tag, id, name, visible text, neighbour text) are all strings that read well in a git diff. The geometric properties (location, area, shape) are the ones to drop: they are viewport-dependent and meaningless in a file reviewed by an agent.

### testRigor-style

[S: https://testrigor.com/docs/language/] Elements are referenced by what a user sees, searched in this order: text, placeholder, value, test-id / id, name, aria-label, CSS class, ML-classified label, hint/title, alt/src, associated label. Ambiguity is resolved by element type (`button "Search"`), by spatial relation to another element (`below "Actions"`), or by containing section. [S: https://testrigor.com/blog/self-healing-tests/] The healing description is marketing-level: it names "role, relative structure, visible text, accessibility labels, and semantic properties" and ML matching, with no algorithm. [A] The transferable idea is disambiguation by a named container ("in the section Billing Address"), which is `refinedBy` again. Proprietary and closed; nothing to reuse.

## 7. What annotation and feedback tools actually store

### SitePing (`@siteping/widget` 0.12.5, MIT, latest publish 2026-11-16)

The most relevant prior art: it solves hover-note's exact problem and documents it. [S: https://siteping.dev/docs/widget/anchoring, the `AnchorData` type in the published `types.d.ts`, and `src/dom/*.ts` embedded in the published sourcemap]

**Stored per annotation:**

| Field | Content |
|---|---|
| `anchorKey` | Value of the closest ancestor's `data-feedback-anchor`, if any |
| `elementId` | The `id`, dropped if longer than 191 chars |
| `cssSelector` | From `@medv/finder` with extra filters: rejects classes matching `^(css\|sc\|emotion\|styled)-` or `^[a-z]{1,3}[A-Za-z0-9]{4,8}$`; rejects ids starting `radix-` or matching `:r1:`; attributes limited to `data-testid`, `data-id`, `role`, `aria-label`. One selector per shadow tree, joined by ` >>> `. |
| `xpath` | XPath to the same element |
| `elementTag` | Tag name |
| `textSnippet` | First 120 chars of text |
| `textPrefix`, `textSuffix` | About 32 chars of text before and after the element in document order |
| `neighborText` | Text of adjacent siblings |
| `fingerprint` | `"childCount:siblingIndex:attrHash"`, the hash (djb2) over `role`, `aria-label`, `type`, `name`, `href`, `src`, `data-testid`, `data-id` |

It also stores the drawn rectangle as percentages of the element's box, not of the viewport.

**Re-find algorithm:**

1. Gather candidates from every strategy at once (key, id, CSS, XPath), up to 16 each.
2. Score each candidate against every stored signal. A signal that was not stored counts neither for nor against.
3. Rank by `strategy prior × verification × visibility`. Priors: key 1.0, id 1.0, CSS 0.95, XPath 0.9, scan 0.85. Hidden elements are ranked far down but not excluded.
4. Only if no selector candidate can beat what a scan could reach, sweep all elements with the stored tag and fully score the best 24. Sweeps are budgeted per pass so many drifted anchors cannot freeze the page.
5. Acceptance is gated on the strongest single signal, not the blend: text similarity ≥0.5 accepts; refuted text needs structural score ≥0.6; scan results need blend ≥0.4. If two candidates are within 0.05 and one contains the other, the inner wins.
6. Confidence below 0.7 is shown as "Approximate position". If nothing verifies, the annotation is orphaned and its marker hidden: "a missing marker is confusing, a lying one is worse".

**Lessons recorded in its source comments** (v1 was a first-match cascade): a wrong-but-text-passing selector hit beat the correct element a scan would have found; the first DOM match won even when it was an invisible responsive twin; elements with no text got no verification at all. Fuzzy matching below 8 characters over a long haystack was "statistically meaningless" and is disabled.

**Size.** Its seven DOM-anchoring source files total 62KB of TypeScript; comment-stripped they gzip to about 8KB before minification, plus finder (measured from the sourcemap). [A] So a full SitePing-equivalent would take roughly half of hover-note's budget once minified. A leaner version is possible by dropping XPath, shadow-DOM chains and the scan budgeting.

**One correction.** A comment in its `fingerprint.ts` says "Inspired by Similo (academic state-of-the-art, 98.8% accuracy)". The Similo paper reports 89% located; 98.8% is the later HybridSimilo figure on a different benchmark (§6).

### Agentation (`agentation` 3.1.2, latest publish 2026-09-22)

[S: package README and `index.d.ts` in the npm tarball; https://github.com/benjitaylor/agentation]

- Stores: `x` (% of viewport width), `y` (px from document top), `element` (a display name), `elementPath` (e.g. `"body > div > button"`), and optionally `selectedText`, `boundingBox`, `nearbyText`, `cssClasses`, `nearbyElements`, `computedStyles`, `fullPath`, `accessibility`, `reactComponents`, `sourceFile`, `attributes`.
- Purpose stated in the README: "captures class names, selectors, and element positions so AI agents can `grep` for the exact code you're referring to".
- [A] This is a description for an agent to read, not an Anchor for re-finding. Markers are positioned by stored coordinates. [U] I grepped the minified bundle and found no step that resolves `elementPath` back to an element, but that check is not exhaustive.
- **Licence is PolyForm Shield 1.0.0, not an open-source licence.** Do not copy code from it. React peer dependency; 136KB gz.

### React Grab (`react-grab` 0.2.0, MIT, latest publish 2026-08-22)

[S: https://github.com/aidenybai/react-grab README] Copies the element's HTML plus its React component stack with source locations, e.g. `<a class="…" href="#">Forgot your password?</a> in LoginForm (at components/login-form.tsx:46:19)`. It is a one-shot copy to the clipboard; nothing is persisted or re-found. React-only (depends on `bippy`); the global build is 99KB gz. [A] The transferable idea is the source location as a signal: for a coding agent, `file:line:col` is a better pointer than any DOM selector, and it is immune to every change in §1 except edits to that file.

### Marker.io

[U] Not documented. Its public docs and SDK README cover captured metadata (URL, browser, console logs, custom data) and screenshots, and say nothing about how a pin is tied to an element or re-found [S: https://github.com/marker-io/browser-sdk; https://help.marker.io/en/articles/5358889-custom-metadata]. [A] It appears to be screenshot-centred, which would make re-finding unnecessary.

## 8. Comparison

Robustness: ✓ survives, ~ partly, ✗ breaks. Ratings are [A].

| Technique | Re-render | Class churn | Copy edit | Layout refactor | Reorder | Cost to resolve | Bytes in core |
|---|---|---|---|---|---|---|---|
| Explicit `data-*` key | ✓ | ✓ | ✓ | ✓ | ✓ | one `querySelector` | ~0 |
| `id` (filtered for generated ids) | ✓ | ✓ | ✓ | ✓ | ✓ | one lookup | ~0 |
| Generated CSS selector, filtered | ✓ | ~ | ✓ | ✗ | ✗ | one `querySelector` | 1–2KB |
| Absolute XPath / tag+index path | ✓ | ✓ | ✓ | ✗ | ✗ | one walk | <0.5KB |
| Role + accessible name (approximate) | ✓ | ✓ | ✗ | ✓ | ✓ | scan by tag | ~0.5KB |
| Element text, fuzzy | ✓ | ✓ | ~ | ✓ | ✓ | scan + edit distance | ~0.5–1KB |
| Surrounding / neighbour text | ✓ | ✓ | ~ | ~ | ✗ | scan | shares the above |
| Attribute fingerprint | ✓ | ✓ | ✓ | ✓ | ✓ | scan | <0.3KB |
| TextQuote (exact + prefix + suffix), fuzzy | ✓ | ✓ | ~ | ✓ | ✓ | text walk + search | 1–1.5KB |
| TextPosition | ✓ | ✓ | ✗ | ✗ | ✗ | text walk | shares the above |
| Geometry (x, y, size) | ~ | ✓ | ✓ | ✗ | ✗ | layout read | not recommended |
| Source location (`file:line`) | ✓ | ✓ | ✓ | ✓ | ✓ | needs a build-time attribute | 0 in core |

## 9. Recommended shortlist

1. **Explicit key first.** Honour an author-placed attribute (a hover-note one, plus the common test-id names) and `id`, filtered for framework-generated values. Best robustness for zero bytes.
2. **Similo-style scoring over a small, readable set of signals:** tag, role, name, own text, neighbour text, stable attributes, tag path. Copy the method and the weights, not the code.
3. **A filtered generated CSS selector** as a fast candidate source and a human-readable hint. Write a small one in-house on finder's penalty model with SitePing's hash filters; finder itself (MIT, ≤2KB) is the fallback if an in-house one proves fiddly, but it would be the library's only dependency.
4. **TextQuote with prefix/suffix, scoped to the anchored element** (`refinedBy` shape), with Hypothesis's scoring. Start with exact match plus a bounded edit-distance check; reach for the Myers algorithm (`approx-string-match`, MIT, ≤1.3KB) only if long-quote fuzzy matching proves necessary.
5. **Rank, do not cascade; threshold plus margin; a visible lost state.** SitePing's v1-to-v2 history is the evidence that cascades return wrong elements.

Rejected: W3C JSON-LD envelope (no benefit without interop), `dom-anchor-text-quote` and Apache Annotator (unmaintained or retired, over budget), XPath strings (redundant with a tag path), geometry (viewport-dependent, noise in git), full accessible-name computation (5KB+), any ML or LLM healing in the core.

## 10. Anchor sketch

Everything in this section is [A]: a proposal, not built or measured.

### Fields

```jsonc
{
  "v": 1,

  // Identity: any one of these, if present, is a near-certain match.
  "key": "pricing-plans",            // author-placed data-hover-note, else data-testid/-test/-qa/-cy
  "id": "checkout-button",           // omitted if it looks generated

  // Element signals: what gets scored.
  "tag": "button",
  "role": "button",                  // explicit role or tag default
  "name": "Place order",             // aria-label, labelledby, label[for], alt, title, else text; ≤80 chars
  "text": "Place order",             // normalized own text, first 120 chars; omitted when equal to name
  "attrs": { "type": "submit", "name": "place", "href": null },  // small allow-list, raw values
  "near": ["Order total", "Back to cart"],   // text just before and just after, ≤32 chars each
  "path": ["main", "form", "div:2", "button"],  // tags from a landmark or keyed ancestor; :n = index among same-tag siblings, omitted when 0
  "css": "form[name=checkout] button[type=submit]",  // filtered generated selector; a hint and a candidate source

  // Text range only: refines the element above.
  "quote": { "exact": "free shipping", "prefix": "Orders over $50 get ", "suffix": " within the EU.", "start": 21 }
}
```

Choices behind this:

- **Plain values, no hashes.** SitePing hashes attributes into a fingerprint; a hash is all-or-nothing and unreadable. Raw values can be compared per attribute, diffed in a PR, and grepped by an agent.
- **`path` is an array, not an XPath string**, so it can be compared by longest common subsequence (Healenium's approach) instead of failing outright.
- **`path` starts at the nearest landmark or keyed ancestor**, not `html`, so wrapper changes above it do not matter.
- **Missing fields are simply absent** and are left out of scoring, neither for nor against.
- **No geometry, no computed styles, no screenshots** in the Anchor.

### Describe (when the Annotation is created)

1. Climb from the target to the nearest ancestor with a key attribute; record `key`. If the target itself has none, keep describing the target and use the keyed ancestor as the start of `path`.
2. Record `id` unless it matches a generated pattern (`:r1:`, `radix-…`, long hex/base36 runs, digits-only suffixes).
3. Record tag, role, name, text (NFC, whitespace collapsed, trimmed; case preserved), the attribute allow-list, neighbour text and path.
4. Generate `css`: per level prefer key attribute, id, stable attributes, word-like classes, tag, then `:nth-of-type`; stop at the first unique path; cap the work (levels and `querySelectorAll` calls).
5. For a text selection: record `exact`, 32 chars of `prefix` and `suffix`, and `start` as an offset **within the anchored element's text**, not the document's. Anchor the element to the closest common ancestor of the range.

### Re-find an element

1. **Candidates.** Collect, without stopping at the first hit: elements matching `key`; `getElementById(id)`; `querySelectorAll(css)` (first 16); the element reached by walking `path`. If the best candidate so far scores below the "confident" bar, add a scan: all elements with the stored `tag` (or role), prefiltered by a cheap text-contains or attribute check, top 24 kept.
2. **Score** each candidate as a weighted mean over the signals the Anchor actually has:

   | Signal | Weight | Comparison |
   |---|---|---|
   | key | 3 | equal |
   | id | 1.5 | equal |
   | tag | 1.5 | equal |
   | name | 1.5 | exact = 1, else edit-distance similarity |
   | text | 1.5 | exact or contains = 1, else similarity on the first 120 chars |
   | near | 1.5 | mean similarity of before/after |
   | role | 0.5 | equal |
   | attrs | 0.5 | fraction of stored attributes that still match |
   | path | 0.5 | LCS length / stored length |

   The 1.5 / 0.5 split is Similo's. Skip fuzzy similarity for strings under 8 characters (exact only), per SitePing.
3. **Decide.**
   - **Found:** best score ≥ 0.8, or `key` matches exactly one element.
   - **Approximate:** best score ≥ 0.5 **and** it beats the runner-up by ≥ 0.1 (unless one contains the other, in which case take the inner). Show it, marked as uncertain.
   - **Lost:** anything else, including two near-equal candidates. Ambiguity is lost, not a coin toss.
   - Additionally require at least one of key, id, name or text to match on its own at ≥ 0.5 when the Anchor has any of them. Structure alone may not carry a match. An Anchor with no text-like signal at all (an icon) may be found on selector + attrs + path, capped at "approximate".
4. **Visibility** only breaks ties: prefer a rendered candidate over a hidden twin, but never discard a hidden one if it is the only match.

The thresholds are starting values taken from the neighbourhood of SitePing's (0.7 approximate, 0.5 text floor) and Healenium's (0.5–0.6 cap). They need tuning against real pages.

### Re-find a text range

1. Re-find the element as above. If found or approximate, take its normalized text; if lost, fall back to the nearest surviving `path` ancestor, then `document.body`.
2. Find `exact` with `indexOf`. One hit: done. Several hits: score each by prefix similarity, suffix similarity and distance from `start` (Hypothesis's 50/20/20/2 weights) and take the best if it leads by a margin.
3. No exact hit: approximate search allowing up to `min(32, exact.length / 4)` errors, scored the same way. This is deliberately tighter than Hypothesis's `length / 2`, because a wrong highlight in a dev tool is worse than an orphan.
4. Map the offsets back to a `Range` with a `TreeWalker` over text nodes.
5. **Degrade in steps, and say so:** quote found → highlight it. Element found but quote gone → keep the Annotation on the element, flagged "text changed", and show the stored `exact` so the reader can see what it referred to. Element lost and quote not found → lost.

### When to declare it lost, and what that means

- Lost is a **runtime state, not a stored one**. The Anchor file is unchanged; the Annotation is listed as unattached with its stored `name`, `text` and `css` so a human or agent can still act on it or re-target it.
- Do not declare lost while the page is still rendering. Resolve on load, then re-resolve on DOM mutations (debounced) and on route changes; report lost only after the page has been quiet for a moment. Similo names this timing problem explicitly.
- Never fall back to stored coordinates.
- Budget scans per pass so a page with many drifted Anchors stays responsive.

### Runtime cost

Happy path: one or two `querySelector` calls and one scoring of a handful of candidates. Worst case: one same-tag sweep plus edit distance over 24 candidates × strings capped at 120 chars, which is well under a millisecond each. Similo measured 4 ms per element scoring every element on the page across 14 properties, which is the pessimistic bound.

### Size estimate

Selector generator ~1KB, describe ~0.7KB, scorer + edit distance ~1KB, text-range walk and quote match ~1KB: **roughly 3–4KB min+gz**. This is an estimate from the measured sizes of comparable code (finder ≤2.0KB, approx-string-match ≤1.3KB, SitePing's fuller version ~8KB unminified-gz). It leaves about 6KB for Targeting, the Thread UI and the Backend client, so it should be confirmed with a spike before the rest of the budget is planned around it.

## Implications for hover-note

1. **The Anchor is a record of several plain signals plus a scorer, not a selector.** This matches the CONTEXT.md definition, which already says selectors and locators are "kinds of Anchor data, not the concept".
2. **Ship a documented opt-in attribute** (and honour existing test-id attributes). It is the only technique that survives a redesign, and in dev/staging builds the people annotating can ask for one to be added. Coding agents can add it themselves when they resolve a Thread.
3. **Build the anchoring code in-house.** Every reusable library is either over budget, unmaintained, retired, or would be the sole dependency. The algorithms are small and well documented; SitePing (MIT) and Hypothesis (BSD-2) are safe to study, Agentation (PolyForm Shield) is not safe to copy.
4. **"Lost" must be a first-class state in the domain model**, alongside a middle "approximate" state. Hypothesis calls it an orphan and gives it a tab; SitePing hides the marker but keeps the feedback. This probably deserves a term in CONTEXT.md and it interacts with the Thread lifecycle.
5. **Store readable values because the Anchor lives in git and is read by agents.** `name`, `text` and `css` double as the description an agent uses to find the code, which is all Agentation and React Grab provide.
6. **Text ranges are an element Anchor refined by a quote**, not a separate kind of Anchor. One code path, and a natural degradation when the text is edited.
7. **Precision over recall.** Use tighter thresholds than Hypothesis and require a margin over the runner-up. In a tool whose notes drive code changes, a note attached to the wrong element is worse than an unattached one.
8. **Anchor text is page content committed to the repo.** That needs a masking attribute or a length cap from day one.

## Open questions

1. **Should a healed Anchor be rewritten?** Healenium updates its stored path after every successful find, which keeps it fresh but, in git, means file churn and merge conflicts from simply viewing a page. Options: never rewrite; rewrite only on an explicit "re-pin"; rewrite only when a human or agent next comments.
2. **Is source location in scope as a signal?** A build-time attribute carrying `file:line` (what React Grab reads from React internals) would be the best pointer for agents and immune to DOM change, but it is framework- and bundler-specific, so it cannot live in a framework-agnostic core. A plugin could write it into the same attribute the core already treats as `key`.
3. **Which attribute name(s) count as `key`?** A hover-note-specific one, the common test-id names, or configurable like Playwright's `testIdAttribute`?
4. **Repeated items.** Rows of a list often share tag, role, attributes and even a test id; only text and index differ. Is "the third row" or "the row that says X" the intended meaning? This may need a per-Annotation choice at Targeting time.
5. **Shadow DOM and iframes.** Open shadow roots need one selector per tree and a path that crosses hosts; closed roots and cross-origin frames are unreachable. In or out for v1?
6. **Page identity.** This document covers finding an element within a page. Which Annotations belong to the current page (URL, route pattern, dynamic segments) is a separate question that decides whether re-finding even starts.
7. **Thresholds and weights are untested.** They are borrowed from tools with different goals. A small corpus of before/after page pairs covering the §1 change types would let them be tuned and would make a regression test for the scorer.
8. **i18n and themes.** A locale switch invalidates every text signal at once. Should an Anchor record the locale it was captured in, or just rely on the structural signals and accept "approximate"?
9. **Is strict W3C compatibility wanted?** If Annotations should ever be exported to other annotation tools, offsets must be in code points and the selector names must be exact. Nothing found here suggests a need.
10. **Unverified items worth a follow-up:** Healenium's scoring constants (secondary sources only), the Kluge & Stocco tuned parameters (abstract only), whether `computedRole` / `computedName` are usable, whether Agentation ever re-resolves `elementPath`, how Marker.io ties pins to elements, GitHub activity for all projects, and true minified sizes.
