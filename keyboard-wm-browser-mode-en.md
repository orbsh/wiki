# Browser usage mode under a tiling WM, full keyboard

> **Languages**: English (translation of the Chinese primary) · [中文](keyboard-wm-browser-mode.md)
> **Status**: landed — tag forest and console (2026-09-05), Rust full-stack rewrite (decided 09-26 / landed 09-28–29, the Python tree is deleted), extension plane (event log + stdio resident host + intercept verdicts + Hook collection, 2026-10-01 to 10-13); not-yet-landed work in §10
> **Trigger**: qutebrowser's modal IME pain → a browser usage mode that is non-modal, minimal UI, full keyboard, externally controllable
> **Position**: from a keyboard UX tool to **information-flow refining** — the browser is the primary inflow of information
> **Upstream reference**: qutebrowser/qutebrowser#3444 (OPEN)
> **Generic content-organization model**: see [the tag forest](tag-forest.md)
> **Implementation details**: the command surface, data model, CDP mapping, protocol shapes and the verified ledger live in the mudra repo's `README.md` and `docs/` — this document only carries the why

## 1. Position: information-flow refining

For most scenarios the browser is the primary inflow of information. This mode is not a cosmetic improvement on "driving the browser from the keyboard"; it turns the browser entry point into an **information-flow pipeline**: **capture → attribute → refine → consume**.

- **Capture**: every page that flows in is received and archived under external control — the controller keeps a record of each page.
- **Attribute**: each page gets **tags** (ownership) and **scores** (importance / urgency / quality).
- **Refine**: filter, sort and settle along those axes; decide what to read, what to read first, what to archive.
- **Consume**: jump, open, export; later, body extraction, full-text search and machine summarization.

The browser turns from "a tool for looking at web pages" into "an entry point that pre-sorts information for you". The premise is **external controllability**: only when every page can be listed, controlled and marked programmatically is classification and refining on top of it possible at all.

What is wanted is **a browser usage mode, not a particular browser**: reproduce qutebrowser's "maximal webview, minimal UI" feel under a tiling WM, while being non-modal, full-keyboard, externally controllable, and consistent with the launcher workflow.

## 2. Hard constraint: almost niri-exclusive

The whole mode rests on two **structural** conditions; remove either and it does not hold:

- **Interface-level fine control**: moving / focusing windows, setting column width and routing workspaces all need IPC actions. niri and hyprland qualify; cosmic-de's window-layer control is insufficient.
- **A workspace model that suits "one instance per workspace / routed by isolation dimension"**. Whether the IPC is good enough is not the criterion; the workspace model is the watershed: hyprland's workspaces are bounded and so is the number of windows a single workspace can lay out, which in practice caps "arbitrary growth of isolation dimensions × multi-window tiling per dimension"; stacking compositors, even with adequate IPC, have no tiling advantage to offer.

niri is the only fitting side because new workspaces are cheap and unbounded, and a single column can stack windows vertically — the developer-tools stack (editor + terminal + debugger in one column) is a normal shape, not an exception.

The conflict between tiling WMs and modern software's preset layouts (shrink the window and responsive layouts break), and why terminal work belongs to a terminal multiplexer — full argument in [Linux desktop workflow](linux-desktop-workflow.md) (Chinese).

## 3. Pain points and the selection

Two pain points:

- **qutebrowser's modal IME has no fix**: it switches between insert and normal mode but exposes no mode-switch hook (upstream #3444; neither autocmd nor a plugin system exists), so the IME eats keys in normal mode while a text field is focused. There is no clean way to get "IME on only in insert mode".
- **The extension layer cannot control enough**: Vimium / SurfingKeys cannot inject into error pages (the `chrome-error` page for 4xx/5xx) or built-in pages, so the control surface is necessarily incomplete.

Rejected:

- **qutebrowser**: the modal IME has no fix and upstream gives no hook.
- **A pure extension layer**: error pages and built-in pages cannot be injected, so the control surface stays incomplete.
- **Nyxt / EVE / uzbl**: fully programmable, but tied to WebKit / Emacs / Lisp — the entry cost and compatibility do not pay off.

Conclusion: **full Chromium + an external controller** (`--app` + CDP + a self-authored extension + a controller with a KV store), each layer doing its own job. The "embed an engine, draw your own tab strip" path is not reproduced — that means maintaining both the engine and a UI; organizing and refining is the tag forest's job.

## 4. Architecture: layer responsibilities and boundaries

- **`--app`, one window per page**: no omnibox, no tab strip — what opens is a clean web page (the WM adds no title bar by default). Minimal UI can only come from `--app`: **CDP cannot touch the browser's own UI chrome** (the tab strip and omnibox are outside its capability), so there is nowhere to shrink chrome. "Tabs" do not live in the browser; the external controller manages them as a tag forest.
- **CDP as the backbone**: every target — error pages and built-in pages included — can be listed and controlled, and `Target.created/destroyed` is the single source of lifecycle truth. That one fact gives both "update the list on close" and crash recovery.
- **The page-side extension (self-authored)**: normal pages govern themselves through it — four-mode keyboard driving, status bar, link hints, scrolling, runtime config — without a daemon round-trip. It is self-authored rather than off-the-shelf: the surface reason is that the off-the-shelf candidates' MV2 builds are silently refused by chromium 139+; the real reason is that the interaction surface (tag capsules, the status bar's data source) must couple with the control surface, and an external extension cannot offer that seam.
- **KV store + web console**: tag-forest organization and refining, isolated instances, window↔process mapping, address recording and filtering, per-site column-width memory. The console is the **only management surface** (the earlier launcher route is dropped; see §8).
- **Extension plane**: the second machine surface beside the control plane; shape and boundaries in §9.

**The engine conclusion**: chromium is the only engine with a mature CDP (`/json` endpoint + WebSocket; targets, windows, navigation and injection are all covered). Alternative engines Ladybird and Servo do have remote control, but both speak the Firefox remote debugging protocol (actor + TCP JSON) and both call themselves incomplete — not CDP, and not yet a mature window/session management surface, so they cannot serve as the controlled engine of a daily driver. The control layer therefore abstracts a `BrowserEngine` interface and waits for whichever engine protocol matures, instead of repackaging now.

## 5. Resource model: instance count follows isolation dimensions

Chromium is **process-per-page**: one renderer per page, whether it is many tabs in one window or one tab per window. So "many `--app` windows in one instance" and "many tabs in one window" cost nearly the same — one browser process + one renderer per page + shared cache and GPU — while "several independent instances" means one browser process and one cache each, noticeably heavier.

Conclusion: **open instances by isolated dimension** — the concurrent instance count equals the number of isolation dimensions, not the number of pages or topics. The price is a few extra browser processes and caches; what it buys is hard isolation of cookies, login state and privacy. Non-isolated content adds no instance and rides many pages inside a shared one.

This arithmetic is the foundation of the storage and window models: the instance is the isolation unit, the page is the content unit, and the two have different cardinalities.

## 6. Address input: one verb is not enough

**The address bar and the command line are one input surface** (2026-10-22, three rounds of decision): `o` / `O` merely type `open ` / `new ` into the same line; once a space appears the candidates switch to "live pages + open history"; Tab moves the selection, a lone candidate completes into the line, Enter takes the highlighted entry (with no match the typed text is used), and **there is no second-level picker**. The command line, the candidate popup above it and the status bar share one appearance — one input surface should not have three faces.

The reason is not that a component disappears: the address bar and the command line answer the same question — "where am I going" — and splitting them into two layers adds a modal switch; once the candidates sit under the input line, a second confirmation carries no new information. Esc cancels: a half-typed string is an intermediate state of input, not an intention, and must not be opened as a URL.

**Two open verbs, not one**: `open` navigates in place (a reading act), `new` opens another page window (a collecting act). Merging them would require deciding every time whether one is reading or opening something new — something only the user knows at that moment, which cannot be guessed. The console page never navigates in place: navigating the management surface away is closing it on oneself.

**Address normalization happens in one place**: both verbs must parse "what the human typed", and two places inevitably drift (scheme completion, bare-word completion written twice). This is a case of pushing the defense line forward, not of saving code.

**Candidate order: live pages before history**. Returning to what is in hand is more common than going to an old place; history is the fallback, not the first choice.

**Closing the current page does not go through fuzzy matching**: `q` closes "the page that said this", named directly by the page side, bypassing the URL-substring close verb — fuzzy matching picks the wrong page in exactly this situation.

## 7. Content organization: the tag forest replaces session

Pages are no longer organized under a single-owner session; they fall into a **tag forest**: many trees, trees do not exclude each other, one value within a tree, no single root (model and degenerate cases in [the tag forest](tag-forest.md)).

- **situation tree** (required, single-select, defaults to inbox): `inbox / work / personal / privacy` — the current context, which also decides isolation.
- **importance / urgency / quality trees** (two levels, ordered by `rank`): scoring *is* a tree, not a separate field — a domain dictionary assigns default scores to leaf tags (a news site high-importance, its user-content subdomain low), with manual overrides.
- **state pipeline tree**: `to-read / reading / distilled / archived` — the refining stage axis, orthogonal to both ownership and weight.
- **topic tree** (plain, multi-select): topic ownership across projects.

The four axes each answer one thing: where (situation), how important (score), how far along (state), about what (topic). Collapsing them into one field grows exceptions immediately — which is precisely why session was replaced.

Isolation instances follow the tag's `isolated` flag: matching content runs in its own profile / cookie jar / process. Always-on content (IM, RSS) takes the pin shape — a dedicated instance kept running long-term, the equivalent of an "app slot".

In terminology **tab → page**: this mode has no tab strip, one page is one window; calling it a "tab" would confuse the content concept with the container concept.

**The page tree is the second axis**: a page opened from another page (CDP `openerId`) forms a tree with the page that opened it. It is orthogonal to the tag forest — the forest speaks of content ownership, the page tree speaks of opening relations, which serve link mining (discovering children that a high-value parent pointed to) and whole-subtree sorting.

## 8. Interaction convergence: one switching layer is enough

**Unified switching is the only switching surface**: every page window is an ordinary niri window and appears directly in Alt+Tab / windowsmru. No second switching scheme is maintained inside the browser — two-layer switching is the legacy of the tab-strip model, and this mode has no tab strip. Many windows are only a side effect; filter them in the launcher when needed.

- **No title prefix, no exclusion**: the earlier plan to prefix `--app` windows so they could be filtered out of generic switchers is dropped. Switching directly with Alt+Tab and filtering in the launcher is already an advantage; introducing a naming convention that must be maintained buys nothing.
- **The management surface converges on the web console**: search and filtering, every page operation, and the tag-forest tree view all live in the console (opened with Mod+Space); addresses, pids and window ids are recorded at spawn, so window titles are never consulted. The launcher side is untouched.
- **The CLI is a scripting source of truth only**, not an interaction surface — interaction and scripting have different optimal shapes, and mixing them dulls both.
- **Close semantics lose nothing either way**: a controller-initiated close deletes the page from the archive (no trace); a WM close or a crash keeps it and marks it closed. An active close is an intention, an external close is an accident; they cannot be handled the same way.
- **Sorting = page tree → workspace**: moving a page together with its whole subtree to a separate workspace is a WM-layer classification act that needs no controller involvement.

## 9. The extension plane: shape and boundaries

The second machine surface beside the control plane. **Shape**: an extension is a **resident subprocess** hosted by the host (stdio pipes), not a federation of network services. The reason: one page event often has several consumers (classification, ad blocking, restyling); if every consumer were a service, the user would be maintaining a service registry. Hosting turns "many consumers" into a process-lifecycle problem.

- **The contract is the dependency**: the wire contract is Induction (the same contract aura's effectors speak; the vocabulary was extracted from aura's BGI), and the controller depends on zero aura crates — portability comes from "one contract", not from a dependency. Inventing our own frame vocabulary and the LSP analogy were both withdrawn: an extension is plugin lineage (capability offered one way, the downstream consumes without exposing), the vocabulary is static and both sides know it natively; the negotiation surface is a pseudo-requirement — LSP's capability negotiation solves alignment between two independently evolving peers.
- **Declared in the script's schema, not in config**: config only carries name → executable path; event subscriptions are carried by the `hello` frame (an out-of-process binary host cannot introspect them). Changing a subscription should not change the config, and it is not a negotiation — it is a static declaration. Delivery fans out by declared kind only; no broadcast, no hidden dispatch.
- **Delivery = the declared `Tier`**: `Hot` + deadline is **intercept semantics** (a verdict before navigation; a timeout is a failure value and the host side fails open — navigation UX first); `Cold` is **observe semantics** (it does not block the producer and may answer later, or not at all). A handler has exactly one of the two shapes.
- **The event log is delivery infrastructure**: the event also lands in the append-only log; consumers hold their own cursor and replay, and an extension restart continues from its own cursor. Checkpoints belong to the consumer; the host does not track progress.
- **Effects stay in the controller**: an extension is a **decision surface** (rules, model calls and state all live there), and effects ride `host:invoke` back into the existing verb surface. An extension never touches CDP and never opens a second browser. The `store` arm is not implemented — extension state belongs to the extension (one home per datum; the host-owned event log is delivery infrastructure, not extension storage).
- **No direct links between extensions**: they share only the event log; to chain them, each reads and writes its own home rather than growing N point-to-point links.
- **The trust model is unchanged**: this machine is the trust boundary; the extension plane, like the control plane, has no identity admission. Cross-node authentication belongs to the federation layer and does not apply to the local surface.
- **Portability**: the same per-language guest runs on both the controller and aura's effectors — the extension plane is one contract with two hosts, not one host's private plugin format.

Protocol details (frame shapes, verdict budget, hook fields, the injection bridge) and the superseded aura + probe federation draft are in the mudra repo's `README.md` and `docs/ADR-extension-protocol.md`.

## 10. Not yet landed

- **The refining pipeline**: body → Markdown + full-text search, naive-Bayes scoring, RSS capture, machine summarization.
- **Error-page routing**: error pages and built-in pages cannot be injected, so key control there can only go through CDP; switching the control surface by target type is not yet measured or defined.
- **Engine abstraction**: `BrowserEngine` exists only as an interface with chromium as the single backend.
- **The extension's `iterate` arms**: empty; touching one returns an explicit error value until a consumer signs up.

The full verified/pending ledger is the "Status & known issues" section of the mudra repo's `README.md`.

## 11. Open questions

- **Which operations error pages still need**: the capability boundary is clear (CDP only), but which of "reload, back, retype the address" are necessary faces has to be defined by measurement.
- **Chrome UI gaps**: download and print were provided by the browser chrome and must be wired separately (via CDP, or dropped).
- **Deepening the tag forest**: the app-level single-select constraint within a tree, multi-hit on isolated tags (currently undefined behaviour), iteration of the domain dictionary and ML assistance; UX details of inbox triage. ML only learns *rules* from manual scores for review — no black box.
- **The hook boundary**: disable / delete cannot recall code already injected into live pages; only a reload re-arms. Whether a "re-arm matching pages" verb is wanted is left until a need appears.
- **Ownership of extension state**: the `store` arm is explicitly not implemented; if a genuinely shared-state extension ever appears, this boundary is reopened.

## Rejected designs

- **qutebrowser's modal input**: no mode-switch hook (#3444), the IME eats keys with no fix.
- **A pure extension layer**: error pages and built-in pages cannot be injected, so the control surface stays incomplete.
- **Nyxt / EVE / uzbl**: tied to WebKit / Emacs / Lisp; entry cost and compatibility do not pay off.
- **The launcher (walker) carrying address input and switching**: there can be only one management surface; two levels split "where am I going" into two decisions (dropped wholesale 2026-09).
- **Title prefix + exclusion from generic switching**: switching with Alt+Tab is already the advantage; a prefix buys nothing.
- **Two-layer switching**: a second in-browser switcher beside Alt+Tab is the legacy of the tab-strip model.
- **Single-owner session**: page ownership is multi-axis; single ownership is only the four axes projected onto one.
- **Embedding an engine / a private runtime**: reproducing "drawing the tab strip inside the engine" means maintaining both the engine and a UI; an artifact that runs only in its own runtime is a demo, not a delivery format.
- **Shrinking / hiding chrome UI over CDP**: outside CDP's capability; minimal UI can only come from `--app`.
- **Extensions as a federation of network services**: several consumers mean maintaining a service registry; hosting turns it into a process-lifecycle problem.

## Cross references

- **[The tag forest](tag-forest.md)**: the multi-axis content model; this document uses one instance of it.
- **[Linux desktop workflow](linux-desktop-workflow-en.md)**: the conflict between tiling WMs and modern software layouts, and why terminal work belongs to a multiplexer.
- **The mudra repo's `README.md`** (with a Chinese twin, `README.zh-CN.md`): every implementation detail — command surface, KV collection layout, CDP mapping, event sync and epoch discipline, column-width memory, the extension plane's implemented shape, the verified ledger.
- **The mudra repo's `docs/`**: `SCHEMA.md` (storage layout), `KEYS.md` (keymap and config), `PANEL.md` (console), `EXTENSIONS.md`, `ADR-*.md` (full decision records, some bilingual).