# Which file layouts keep threaded notes git-friendly and agent-readable?

Researched 2026-11-17. Vocabulary follows `CONTEXT.md` (Annotation, Anchor, Thread, Comment, Backend).

**How to read this.** Each claim is tagged:

- **[S]** sourced: the linked primary source says it (docs, spec, or source code read at the commit given).
- **[M]** measured here: produced by a throwaway script in this session (method described inline; scripts not committed).
- **[I]** inference / synthesis: my reasoning from the above. Treat as opinion.

Source code was read (never run or installed) from shallow clones at: `kunchenguid/lavish-axi@d0c1667`, `benjitaylor/agentation@0e3236e`, `MrLesk/Backlog.md@373a16d`, `aidenybai/react-grab@ea4bbec`, `kunchenguid/axi@c76bcf2`, `toon-format/spec@009074d`.

## TL;DR

1. **[I]** Nothing in the precedents stores *threads* as plain repo files and also merges concurrent same-thread edits cleanly, except layouts where **every write creates a new file** (changesets, git-issue) or an **op-log merged by a tool** (git-bug, git-appraise, beads/Dolt). One-file-per-item Markdown (Backlog.md) is the most readable but conflicts when two branches append to the same thread.
2. **[M]** In a local git experiment, only "file per Comment" and "JSONL op-log + `merge=union`" merged all four concurrent-edit scenarios cleanly. `merge=union` is reportedly **not honoured by GitHub's PR merge button**, so only file-per-Comment is conflict-free everywhere.
3. **[S]** TOON and AXI are about **CLI output**, not storage: AXI says "Convert to TOON at the output boundary — keep internal logic on JSON". **[I]** Store boring JSON/Markdown; print TOON from the CLI.
4. **[I]** Recommended bridge: a small standalone local server started by the CLI, bound to `127.0.0.1`, with Host allowlist + Origin check, writing only into a fixed directory by validated id. This is what lavish-axi, Agentation and Backlog.md's web UI all do.
5. **[I]** On staging there is no repo: show committed Annotations read-only from a static file, keep new ones in the browser, and export/import; a remote Backend is the upgrade path.

Confidence: high on what each tool does today (read from source/docs); medium on merge behaviour beyond the four scenarios tested; **low on token counts** (no tokenizer available in this sandbox, bytes and a crude proxy only).

---

## 1. Precedents

| Tool | Where the data lives | Format | Id scheme | Concurrent edits from two branches | Deletion / resolution |
|---|---|---|---|---|---|
| **git-bug** | Not in the working tree. A chain of commits per entity under `refs/<namespace>/<id>`; each commit's tree has an `ops` blob | JSON array of operations ("OperationPack") | hash of the first operation, shown truncated to 7 chars | Op-based CRDT-style: tool creates a merge commit in the entity DAG, orders ops by Lamport clock then pack id. No text conflicts by design | Status is an operation (`"status": 1`); nothing is deleted |
| **git-appraise** | `git-notes` under `refs/notes/devtools/*` | One JSON object per line | Comments attach to a commit; threading via fields in the comment schema | Single-line-JSON was chosen so notes "can be automatically merged using the `cat_sort_uniq` strategy" | Latest-timestamp request wins; append-only |
| **beads** (today) | Dolt SQL database in `.beads/` (gitignored), synced via `refs/dolt/data` on the git remote | SQL; `.beads/issues.jsonl` is an optional export | random-UUID-derived short hash `bd-a1b2`, grows 4→6 chars; children `bd-a1b2.1` | Dolt "cell-level merge". JSONL explicitly "is not the source of truth" and "cannot safely reconcile deletes" | Soft-delete tombstone fields (`deleted_at`, `deleted_by`, `delete_reason`, status `tombstone`) |
| **Backlog.md** | `backlog/tasks/<prefix>-<n> - <Title>.md`, one file per task; `archive/`, `completed/`, `drafts/` dirs | Markdown + YAML front matter; body sections fenced by HTML comment markers (`<!-- COMMENTS:BEGIN -->`) | Sequential `TASK-12`; subtasks `TASK-12.1` | Plain git text merge. Sequential ids force it to scan other branches/worktrees to allocate and to resolve "latest state across branches" | `status:` in front matter; archive = move file to another dir; delete releases the id |
| **changesets** | `.changeset/<random-name>.md`, one file per change intent | Markdown + YAML front matter | "random human readable names by default to avoid collisions" | Each write is a new file, so no conflict | Files are consumed and deleted by `changeset version` |
| **git-issue** | `.issues/issues/xx/xxxx…/` one **directory per issue**, in its own git repo | One small plain-text file per field (`description`, `tags`, `assignee`…), `comments/` dir with **one file per comment** | SHA of the commit that created the issue/comment | Different fields and different comments are different files, so they merge independently | Field files; tags (open/closed) one per line |
| **adr-tools / MADR** | `doc/adr/NNNN-title.md` | Markdown | Sequential `0001` | Known problem: two PRs both create `0006` with no textual conflict (silent duplicate) | Never deleted; status line "Superseded by …" links old and new |
| **Agentation** | Browser `localStorage`; with the MCP server, SQLite at `~/.agentation/store.db` | SQLite rows | server-generated | n/a (not in git) | status field per annotation |
| **lavish-axi** | `~/.lavish-axi/state.json` (one JSON file, rewritten whole) | pretty JSON | session keyed by canonical HTML file path | n/a (not in git) | session ends |

Sources, in row order:

- git-bug **[S]**: <https://github.com/git-bug/git-bug/blob/master/doc/design/data-model.md> ("Instead of storing the final bug data directly, we store a series of edit `Operation`s"; ordering algorithm; ids).
- git-appraise **[S]**: <https://github.com/google/git-appraise> README, "Metadata" section. `cat_sort_uniq` is defined in <https://git-scm.com/docs/git-notes> ("concatenating the local and remote versions… sorts the resulting lines, and removes duplicate lines").
- beads **[S]**: <https://beads.gascity.com/architecture>, <https://beads.gascity.com/core-concepts/sync-concepts>, <https://beads.gascity.com/core-concepts/hash-ids>, <https://github.com/steveyegge/beads/blob/main/docs/ARCHITECTURE.md> (tombstone fields; "Hash-based IDs derived from random UUIDs"). A maintainer states "in 1.0.0 Beads no longer persists issues in jsonl and committed to the Git repo, it uses Dolt exclusively": <https://github.com/gastownhall/beads/issues/3135>.
- Backlog.md **[S]**: repo at the commit above — `backlog/tasks/readme.md`, a real task file (`back-644 …md`) for the front matter and marker-fenced `## Comments` section, `src/core/backlog.ts` (`generateNextId`, "worktrees share the task ID namespace before their files are committed"), `src/core/cross-branch-tasks.ts`, README (archive/delete semantics): <https://github.com/MrLesk/Backlog.md>.
- changesets **[S]**: <https://changesets.dev/faq>, <https://github.com/changesets/changesets/blob/main/docs/detailed-explanation.md>.
- git-issue **[S]**: <https://github.com/dspinellis/git-issue> README, "Internals".
- ADR **[S]**: <https://github.com/npryce/adr-tools> (numbered files, `-s` supersede); numbering collision discussed in <https://github.com/adr/madr/issues/28>.
- Agentation **[S]**: `mcp/src/server/store.ts` ("By default uses SQLite (~/.agentation/store.db)"), toolbar prop doc "If not provided, uses localStorage only": <https://github.com/benjitaylor/agentation>. So Agentation is **not** a precedent for repo files.
- lavish-axi **[S]**: `src/paths.js` (`stateDir()` → `~/.lavish-axi`, `state.json`), `src/session-store.js` (`writeFile(this.file, JSON.stringify(state, null, 2))`): <https://github.com/kunchenguid/lavish-axi>.

### What the precedents teach **[I]**

- **Two families.** (a) Tools that want conflict-free merging move the data *out of the working tree* and merge with their own code (git-bug refs, git-appraise notes, beads/Dolt). (b) Tools that want plain reviewable files accept git's line merge and shape files so that it rarely bites (Backlog.md, changesets, git-issue, ADRs).
- **beads is a cautionary tale for hover-note's default.** It started as JSONL-in-git and moved to a database with its own sync ref; the cost is a second sync channel users find confusing (issue #3135 above). hover-note's "files in the repo by default" goal rules that route out. I could not verify from a primary source how the pre-Dolt JSONL merge worked (custom merge driver etc.); do not rely on memory of it.
- **Sequential ids do not survive branches.** Backlog.md pays for them with cross-branch/worktree scanning; ADR numbering silently duplicates. Random ids (beads, changesets) need no coordination.
- **"New file per write" is the only plain-file shape that never conflicts.** changesets and git-issue both use it.
- **Resolution is a status, deletion is a tombstone or a move.** Nobody hard-deletes in the normal flow; beads says JSONL import "cannot infer that records absent from an export were deleted".

### Merge experiment **[M]**

Method: for each layout, a base commit with 5 Annotations, two branches each making one change, then `git merge` (git 2.43.0, default `ort`). For op-logs the merged file was replayed and checked. Four scenarios only; this is a smoke test, not a proof.

| Layout | both add a new Annotation | edit different Threads | both comment on same Thread | A resolves, B comments (same Thread) |
|---|---|---|---|---|
| single pretty JSON file | CONFLICT | clean | CONFLICT | clean |
| JSONL snapshot, 1 line per Annotation | CONFLICT | clean | CONFLICT | CONFLICT |
| JSONL snapshot + `merge=union` | clean | clean | clean but **duplicate ids** | clean but **duplicate ids** |
| file per Annotation, pretty JSON | clean | clean | CONFLICT | clean |
| file per Annotation, Markdown + front matter | clean | clean | CONFLICT | clean |
| single JSONL op-log (append) | CONFLICT | CONFLICT | CONFLICT | CONFLICT |
| single JSONL op-log + `merge=union` | clean, replay ok | clean, replay ok | clean, replay ok | clean, replay ok |
| JSONL op-log per Annotation + `merge=union` | clean | clean | clean | clean |
| directory per Annotation, file per Comment/event | clean | clean | clean | clean |

Caveats on `merge=union`:

- **[S]** Git's own docs: union "tends to leave the added lines in the resulting file in random order and the user should verify the result. Do not use this if you do not understand the implications." <https://git-scm.com/docs/gitattributes>. So an op-log merged this way must be order-independent (sort by timestamp/id on read).
- **[S, secondary]** GitHub's PR mergeability check and merge button reportedly ignore `.gitattributes` merge drivers including `union`: long-running community request <https://github.com/orgs/community/discussions/9288> and a 2026 measured report <https://github.com/commandprompt/pgcolumnar/issues/1116>. I found no GitHub documentation confirming or denying it, and did not test it. If true, union layouts still show "This branch has conflicts" on GitHub and need a local merge.
- **[I]** A snapshot file with union silently keeps both versions of a changed line, which is worse than a conflict. Union is only safe for append-only logs.

## 2. Token cost and ergonomics for an agent

### What TOON is, and what its own numbers say **[S]**

- TOON is "a compact, human-readable encoding of the JSON data model"; "Its sweet spot is uniform objects… For deeply nested or non-uniform data, JSON may be more efficient." <https://github.com/toon-format/toon>
- Spec v4.4, dated 2026-10-08, status **Working Draft**: <https://github.com/toon-format/spec/blob/main/SPEC.md>. Tabular form requires every element to have the same keys and only primitive (or uniformly nested) columns; "containing any array value… disqualifies the whole array" (§9.3) and falls back to list form (§9.4).
- Project benchmark (4 models, 244 questions): TOON 72.2% accuracy at 2,474 tokens vs pretty JSON 71.4% at 4,308 and compact JSON 69.0% at 2,892; headline "42.6% fewer tokens". The README notes that overlapping confidence intervals mean accuracy differences are "not statistically meaningful". These are the project's own benchmarks, not independent ones. <https://github.com/toon-format/toon#benchmarks>
- "When Not to Use TOON": non-uniform structures ("compact JSON often wins outright") and semi-uniform arrays.

**[I]** An Annotation with a nested `comments` array is exactly the non-uniform case: the list of Annotations cannot be one table unless Comments are split into a second table.

### AXI principles **[S]**

From <https://github.com/kunchenguid/axi> (`principles.yaml`, and the skill at `.agents/skills/axi/SKILL.md`):

1. Token-efficient output: TOON on stdout, "~40% token savings over JSON". "Convert to TOON at the output boundary — keep internal logic on JSON."
2. Minimal default schemas: 3–4 fields per list item.
3. Content truncation with size hints and a `--full` escape hatch.
4. Pre-computed aggregates (counts, statuses) to save round trips.
5. Definitive empty states ("0 results", not empty output).
6. Structured errors and exit codes: idempotent mutations, no interactive prompts, fail loud on unknown flags.
7. Ambient context: opt-in session hooks first, then an on-demand skill.
8. Content first: no-argument run shows live data, not help.
9. Contextual disclosure: suggest next steps after each output.
10. Consistent per-subcommand help.

AXI's benchmark claims (gh-axi 100% success at $0.050 vs `gh` 86% at $0.054, Claude Sonnet 4.6) are the author's own and for other domains; the README says other models "may respond differently to output format".

### Sample comparison **[M]**

20 synthetic Annotations with 47 Comments (route, status, Anchor with selector/testid/text/source, Comments with id/author/kind/timestamp/body). Same data in every row except the last.

| Representation | Bytes | vs pretty JSON | Lines | Proxy tokens* |
|---|---|---|---|---|
| JSON pretty, one file | 18,792 | 100% | 613 | 6,519 |
| JSON compact, one file | 12,878 | 69% | 1 | 5,907 |
| JSONL snapshot (1 line per Annotation) | 12,860 | 68% | 20 | 5,899 |
| JSONL op-log (1 line per op) | 14,410 | 77% | 73 | 6,979 |
| Markdown + front matter (20 files, summed) | 10,804 | 57% | 428 | 4,086 |
| TOON nested (list form, Comments tabular per item) | 12,190 | 65% | 248 | 4,187 |
| TOON normalised (2 tables: annotations, comments) | 9,508 | 51% | 69 | 3,725 |
| TOON list view, 6 fields, body truncated (lossy) | 2,211 | 12% | 21 | 782 |

\* **Not real token counts.** No tokenizer could be installed in this sandbox, so "proxy tokens" counts word runs, digit groups and punctuation marks. Use the byte column and the ordering, not the absolute numbers. The TOON was hand-encoded to spec §7.2/§9/§10, not produced by the reference encoder. The Markdown figure excludes the per-file overhead a tool call adds when an agent opens 20 files.

Reading the table **[I]**:

- The big win is not the format but **schema and truncation** (AXI principles 2 and 3): the list view is ~8× smaller than any full encoding. An agent should start from `hover-note list` and fetch one Thread at a time.
- Among lossless encodings, normalised TOON and Markdown are roughly half of pretty JSON, consistent with TOON's own ~40% claim. Nested TOON barely beats compact JSON (65% vs 69% of bytes), as TOON's docs predict for non-uniform data.
- JSON key repetition is the cost of JSONL; an op-log repeats even more.

### Read/write ergonomics **[I]**

| Format | Agent reads | Agent writes without the CLI | Human reads a diff |
|---|---|---|---|
| Single JSON | must load everything | must re-serialise or do a careful in-place edit; escaping errors break the whole file | noisy |
| JSONL | `grep` an id gives the full record; long lines | append a line (easy), edit a line (awkward); one bad line is isolated | poor: one long changed line |
| Markdown + front matter | natural; one file = one Thread | append text at the end; front matter edits are simple but YAML is fragile | best |
| File per Comment | needs a glob or the CLI to assemble a Thread | create a new file: no parsing, no editing | good: each Comment is an added file |
| TOON as storage | fine | row/length counters (`[N]`) must be kept in sync by hand; spec is still a Working Draft | fine |

TOON's `[N]` length guard is useful for catching truncated *output*, but it makes hand-appending to a *stored* file a two-place edit. Combined with AXI's own advice, that argues against TOON on disk.

## 3. How a browser page can write into repo files in development

| Approach | Works with | Extra setup | Main risks | Precedent |
|---|---|---|---|---|
| **Dev-server middleware plugin** | Only the bundler it is written for | Add a plugin to the config | Inherits the dev server's exposure (`--host`, allowed hosts, CORS); one plugin per bundler | Vite `configureServer`; `/__open-in-editor` (Vite) and `/__nextjs_launch-editor` used by React Grab |
| **Standalone local server started by a CLI** | Any stack, including non-JS backends | Run one command; page needs the port | Cross-origin from the page, so CSRF/DNS-rebinding must be handled explicitly | lavish-axi, Agentation MCP server (port 4747), Backlog.md `backlog browser`, stagewise CLI proxy (port 3100) |
| **File System Access API** | Chromium only | User picks the directory, re-grants permission | No server at all; but handle is per-origin, needs a click, not in Firefox/Safari | vscode.dev |
| **Export / download / clipboard** | Everything | Manual step | None beyond user error | Agentation's default "copy structured output" |

Facts behind the table:

- **Vite [S]**: `configureServer` is "the most common use case… adding custom middlewares to the internal connect app" and "is not called when running the production build" <https://vite.dev/guide/api-plugin>. Vite's dev server had a real localhost vulnerability: before 6.0.9/5.4.12/4.5.6, "any websites were able to send any requests to the development server and read the response" via permissive CORS, unchecked WebSocket Origin, and unchecked Host header (DNS rebinding); the advisory stresses it "even applies to users that only run the Vite dev server on the local machine" <https://github.com/vitejs/vite/security/advisories/GHSA-vg6x-rcgg-rjx6>. The fixes are the model to copy: `server.allowedHosts` (localhost, `.localhost`, IPs by default) and a default CORS origin regex limited to localhost/127.0.0.1/::1 <https://vite.dev/config/server-options>. Path-traversal advisories mostly bite when the server is exposed with `--host` (e.g. <https://github.com/vitejs/vite/security/advisories/GHSA-4w7w-66w2-5vf9>).
- **webpack / Next.js**: not researched from primary sources in this pass. From memory (unverified): webpack-dev-server has `setupMiddlewares`; Next.js has no public dev-server middleware plugin hook, so a Next integration would be a user-added dev-only route handler or the standalone server.
- **lavish-axi [S, source]**: binds `127.0.0.1` by default (`bindHost` → `LOOPBACK_HOST`); "Concrete listen addresses. Never includes 0.0.0.0 / ::" (`src/paths.js`). `src/server.js` has (1) a **Host-header allowlist** as the "DNS-rebinding guard", with the comment that a same-origin check "stops classic cross-origin CSRF but NOT DNS rebinding"; (2) a second guard rejecting mutating requests with a present, non-matching `Origin`/`Referer` (403 "cross-origin request rejected"); (3) path confinement: an artifact route "may read: the canonical HTML file's own directory" using `realpath` (`src/artifact-path.js`); (4) artifacts served with a `sandbox` CSP so they get an opaque origin. Browser→agent delivery is **long polling by the CLI** (`lavish-axi poll`), not the agent watching files.
- **Agentation [S, source]**: `mcp/src/server/http.ts` — unless started with a non-loopback `--host`, it rejects non-loopback socket addresses ("Remote access is disabled…") and non-local `Host` headers ("Host not allowed"); `cors.ts` allows only local dev origins by default (localhost, `*.localhost`, 127/8, RFC1918 ranges, `.local/.test/.internal`) and requires `AGENTATION_CORS_ORIGINS` for anything else. Note the default origin list includes private LAN addresses, so another machine's page on the LAN counts as "local" for CORS (the loopback socket check still applies).
- **Backlog.md [S, source]**: README: the web UI "listens on `127.0.0.1` and is not reachable from other devices on the LAN or VPN"; a non-loopback hostname prints "WARNING: INSECURE REMOTE ACCESS… Anyone who can reach this server can read and change Backlog data… without authentication" (`src/utils/browser-host.ts`).
- **stagewise [S, docs]**: the (pre-IDE) CLI was "a development proxy server that sits between your browser and your development application" on port 3100, injecting the toolbar into HTML documents and hosting a WebSocket endpoint for the agent <https://stagewise.io/docs/advanced-usage/cli-deep-dive>. The repo is now an Electron-style "agentic IDE" with its own browser <https://github.com/stagewise-io/stagewise>. I did not read its security handling.
- **React Grab [S, source/docs]**: the current tree is clipboard-first (copy context, paste into the agent) and its "open in editor" action calls the dev server's own endpoint <https://github.com/aidenybai/react-grab/blob/main/packages/react-grab/docs/architecture.md>. I found no local bridge server in the current packages; an earlier "React Grab for Agents" local relay is described only in a secondary blog post and is unverified.
- **File System Access API [S]**: `showDirectoryPicker()` is "not Baseline", secure-context only, and "Transient user activation is required" <https://developer.mozilla.org/en-US/docs/Web/API/Window/showDirectoryPicker>. Chrome may refuse sensitive folders and, since Chrome 122, offers "Allow on every visit" persistent permission when the handle is kept in IndexedDB <https://developer.chrome.com/blog/persistent-permissions-for-the-file-system-access-api>. Firefox and Safari expose no picker (secondary: <https://openpwa.net/reference/capabilities/file-system-access/>).
- **Local Network Access [S]**: Chrome 142 launched a permission prompt for requests from a public site to local/loopback addresses; "The ability to request this permission is restricted to secure contexts" <https://developer.chrome.com/blog/local-network-access>. A page served *from* localhost does not trigger it <https://developer.chrome.com/docs/modern-web-guidance/guides/security/local-network-access>. **[I]** So a staging page (public https) calling a developer's `localhost` bridge gets a browser permission prompt in Chrome; the bridge is a dev-machine feature, not a staging feature.

### Security checklist for the bridge **[I]**, each item copied from a tool above

1. Bind `127.0.0.1` only; never `0.0.0.0` by default; loud warning if the user overrides (lavish-axi, Backlog.md).
2. Validate the `Host` header against an allowlist to stop DNS rebinding (Vite `allowedHosts`, lavish-axi, Agentation).
3. Reject mutating requests whose `Origin` is not an allowed local dev origin; no `Access-Control-Allow-Origin: *` (Vite advisory, Agentation `cors.ts`). Require a JSON content type so writes always need a CORS preflight.
4. Never accept a path from the browser. Accept Annotation/Comment ids, validate against a strict pattern, and build the path server-side inside one fixed directory; `realpath`-check the result (lavish-axi).
5. The writer must not exist in production bundles (Vite: `configureServer` is not called in build; React Grab guards on `NODE_ENV === "development"`).
6. Comment bodies are untrusted text written by whoever can reach the page and later read by a coding agent: a prompt-injection path into the repo. Keep the bridge off on any shared/staging host and mark author/kind on every Comment.

## 4. Staging and deployed builds (no repo to write to)

Sourced facts: Agentation's default with no server is `localStorage` plus copy-to-clipboard structured Markdown (README and toolbar prop doc). lavish-axi's hosted sharing is "explicit and opt-in" via a third-party service (README). Chrome gates public→localhost calls behind a permission (above). `CONTEXT.md` already names a remote Backend ("a remote store such as SitePing").

Options **[I]**:

| Option | Reviewer effort | Infra | Notes |
|---|---|---|---|
| Read-only: ship the committed Annotations as a static JSON asset in the staging build | none | none | Staging reviewers at least *see* existing Threads |
| Browser-local drafts (`localStorage`) + export file / copy; developer runs `hover-note import` | one manual hand-off | none | Agentation's default behaviour; drafts are per-browser and lost if cleared |
| Remote Backend (hosted store) behind the same Backend interface | none | a service + auth | The real answer for teams; out of scope for the file layout |
| Serverless endpoint that commits or opens a PR via the git host's API | none | a token-holding function | Puts repo-write credentials behind a public page; needs auth and abuse limits |
| Developer's local bridge reached from the staging page | permission prompt | none | Works only for the developer's own browser; LNA prompt in Chrome; mixed trust |

Recommendation **[I]**: static read-only + local drafts + export/import as the zero-infrastructure default; remote Backend as the documented upgrade. Imports should go through the same id scheme so re-importing the same export is idempotent (AXI principle 6).

## 5. Candidate layouts

Shared choices for all three **[I]**: directory `.hover-note/` at the repo root; random ids, not sequential (beads, changesets vs. Backlog.md/ADR problems); a UTC-timestamp prefix so filenames sort chronologically; "resolved" is a status, not a deletion.

### A. One Markdown file per Annotation (Backlog.md shape)

`.hover-note/annotations/20261117-k3f9.md`

```markdown
---
id: 20261117-k3f9
status: open            # open | resolved
route: /pricing
anchor:
  selector: "main .pricing-card button.primary"
  testid: pricing-cta
  text: "Start free trial"
  source: "src/components/PricingCard.tsx:42"
created: 2026-11-17T09:27:00Z
---

### ana (human) 2026-11-17T09:27:00Z ^c1a2

Button label wraps onto two lines at 375px.

### claude-code (agent) 2026-11-17T10:02:00Z ^c7f0

Fixed in PricingCard.tsx:42, added `white-space: nowrap`.
```

- For: best human diff and best raw agent read (one file = one Thread); smallest lossless size after normalised TOON; strong precedent.
- Against **[M]**: two branches commenting on the same Thread conflict (both append at EOF). Needs a Markdown-section parser; a Comment body containing `### ` or `---` needs escaping.
- Deletion: delete the file; git reports modify/delete if the other branch touched it, which is the right outcome.

### B. One directory per Annotation, one file per Comment (git-issue / changesets shape)

```
.hover-note/annotations/20261117-k3f9/
  annotation.json
  20261117T092700Z-c1a2.md
  20261117T100200Z-c7f0.md
```

`annotation.json`

```json
{
  "id": "20261117-k3f9",
  "status": "open",
  "route": "/pricing",
  "anchor": {
    "selector": "main .pricing-card button.primary",
    "testid": "pricing-cta",
    "text": "Start free trial",
    "source": "src/components/PricingCard.tsx:42"
  },
  "created": "2026-11-17T09:27:00Z"
}
```

`20261117T100200Z-c7f0.md`

```markdown
---
author: claude-code
kind: agent
---
Fixed in PricingCard.tsx:42, added `white-space: nowrap`.
```

- For **[M]**: clean in all four merge scenarios with no `.gitattributes`, so it should also be clean on GitHub. Adding a Comment is "create a file": no parsing or in-place edit for agents or the bridge. Status lives in one small pretty-JSON file, so "A resolves, B reopens" is a real conflict and "A resolves, B comments" is not.
- Against: ~3.4 files per Annotation in the sample (67 files for 20); reading a Thread raw needs a glob; front matter repeated per Comment. Deleting the directory on one branch while another adds a Comment leaves an orphan Comment file with no `annotation.json` — the reader must treat that as "deleted with late reply" rather than crash.
- Thread order comes from filenames, i.e. wall clocks on different machines; fine for display, not for anything stricter.

### C. One JSONL op-log per Annotation, `merge=union` (git-appraise / git-bug shape, in the working tree)

`.hover-note/annotations/20261117-k3f9.jsonl`

```jsonl
{"op":"create","id":"20261117-k3f9","at":"2026-11-17T09:27:00Z","route":"/pricing","anchor":{"selector":"main .pricing-card button.primary","testid":"pricing-cta","text":"Start free trial","source":"src/components/PricingCard.tsx:42"}}
{"op":"comment","id":"c1a2","at":"2026-11-17T09:27:00Z","author":"ana","kind":"human","body":"Button label wraps onto two lines at 375px."}
{"op":"comment","id":"c7f0","at":"2026-11-17T10:02:00Z","author":"claude-code","kind":"agent","body":"Fixed in PricingCard.tsx:42, added `white-space: nowrap`."}
{"op":"status","at":"2026-11-17T11:15:00Z","by":"ana","status":"resolved"}
```

plus `.hover-note/.gitattributes`: `*.jsonl merge=union`

- For **[M]**: clean local merges in all four scenarios; append-only writes; full history of status changes; one file per Thread; `grep`-friendly.
- Against: state must be computed by replay, sorted by `at` then `id` because union does not preserve order **[S]**; GitHub PRs reportedly ignore `union` and will show conflicts for same-Thread appends; the largest encoding in the sample; worst human diff; concurrent status ops need a tie-break rule (last `at` wins).

### Recommendation **[I]**

**B** if concurrent same-Thread writes from different branches/worktrees are expected (several agents in worktrees, reviewers on a staging branch) — it is the only plain-file layout that was conflict-free without relying on merge configuration. **A** if that is judged rare and readability matters most; it is the least code and the best raw read. **C** only if an audit trail of status changes is a requirement. Whichever is chosen, keep one module that turns the directory into an in-memory list so the on-disk shape can change later.

### Recommended bridge **[I]**

A **standalone local server started by the CLI** (`hover-note serve`), with the security checklist in §3. Reasons: it is the only option that is framework-agnostic (the project's stated goal) and it is the pattern all three closest tools converged on. Write the request handler with the plain Node `(req, res, next)` signature so a Vite plugin can later mount the same handler via `configureServer` (same origin, so no CORS and no port to configure) without a second implementation. File System Access API is not a fit as the default: Chromium-only, needs a click and re-granting, and it does nothing for the agent side. Export/import stays as the fallback and as the staging path.

For the agent→browser direction, lavish-axi uses CLI long polling **[S]**; with files as the Backend the simpler equivalent is the server watching `.hover-note/` and pushing changes to the page **[I]**.

## Implications for hover-note

- Store JSON/Markdown; emit TOON only from the CLI, with a 4–6 field list view, truncated bodies, a count line, and an explicit "0 open annotations" state (AXI 1–5).
- Use random, time-prefixed ids for Annotations and Comments. Do not use sequential numbers.
- Decide between layouts A and B on one question: how often will two branches write to the same Thread? The experiment says B never conflicts and A conflicts exactly then.
- Do not depend on `merge=union` or a custom merge driver for correctness: drivers need per-clone git config, and GitHub's merge button reportedly ignores them.
- Resolution is `status: resolved`. Offer deletion as an explicit prune command, and define what an orphan Comment means.
- The bridge is a loopback-only CLI server with Host and Origin checks that accepts ids, never paths, and is absent from production builds.
- Staging: read-only static Annotations + local drafts + export/import by default; the Backend interface should make a remote store a drop-in.
- Every Comment carries `author` and `kind` (human/agent); Comment text reaching an agent from a shared environment is an injection surface.

## Open questions

1. How common are concurrent same-Thread writes across branches in the target workflow? (Decides A vs B.)
2. Does GitHub's PR merge really ignore `merge=union` today? Reported by users, not confirmed by GitHub docs, not tested here. Same question for GitLab.
3. Real token counts: rerun the §2 sample through a real tokenizer and the reference TOON encoder; the numbers here are byte counts and a proxy.
4. Should Annotations be grouped by route/page (one directory per page) so an agent working on one page reads only its Threads? No precedent examined does this; untested for merge behaviour.
5. Anchors change when the page changes. Is re-anchoring an in-place edit of `annotation.json` (conflict-prone if two branches re-anchor) or an appended op?
6. Should `.hover-note/` be committed on feature branches at all, or kept on a dedicated branch/ref (the git-bug/beads route) to keep PR diffs clean? The latter contradicts "files in the repo by default" but removes most merge concerns.
7. Next.js and webpack-dev-server integration points were not verified from primary sources.
8. Single-machine concurrency (browser bridge and agent writing at once): atomic write via temp file + rename is assumed sufficient; not examined in any precedent's source here.
9. Stagewise's and React Grab's earlier localhost bridges were not verified in source; only current docs/trees were read.
