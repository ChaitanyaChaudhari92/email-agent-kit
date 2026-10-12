# Figma mode

Source of truth = node data. The rendered Figma PNG is only a validation reference. All values you put in the spec (px, hex, line-height, padding) come from nodes, never from eyeballing.

## 1. Discover (one call: `GET /v1/files/:key?depth=2`)
Key from `/design/:key/`, `/file/:key/` or `/branch/:branchKey/` (use the branch key). `node-id=12-34` means `12:34`. Record file name, version, pages, top-level frames (including frames inside top-level SECTIONs).
- `node-id` → FRAME = that template; SECTION/PAGE = its qualifying frames. Otherwise all qualifying frames on all non-excluded pages.
- Exclude pages matching `/cover|component|library|archive|playground|old|deprecated|icons?|assets?|style ?guide|design ?system|template ?kit/i`.
- **Qualifying frame:** visible FRAME/COMPONENT, width 280–800, height ≥200, ≥1 text node, name not starting `_` or `.`, name not matching `/(^|\W)(ref|reference|archive|old|deprecated|wip|draft|notes?|cover|legend|spec|moodboard|annotation)s?(\W|$)/i`. Zero qualifying and no node-id → ask (instructions §2).
- **Build every qualifying frame.** If a limit stops you, checkpoint and report done vs pending.
- **Variants:** strip `mobile|mob|sp|375|390|360|320|dark|light|desktop|web|600|640|v\d+` and punctuation; equal names = one template. Width ≤480 = mobile variant (drives mobile overrides); name contains `dark` = dark variant (drives `tokens.dark`). A variant with no base becomes the base (log).
- **Container width W:** 480–700 → `meta.width = W`. >700 (a web design) → 600 and scale all linear values by 600/W, flag reduced fidelity. <480 with no desktop frame → 600, values 1:1, fluid.
- Template name: cleaned frame name (title-case); if generic (`Frame 12`, `Email`, `Untitled`) prefix the file name.

## 2. Extract (sequential, ≥1 s apart; never parallel)
1. Nodes: `GET /files/:key/nodes?ids=<≤5 roots incl. their mobile/dark frames>`; save each response to disk immediately; on timeout halve the chunk.
2. Image fills: one call `GET /files/:key/images`.
3. Variables (optional): `GET /files/:key/variables/local`; ignore 403/404.
4. Renders: `GET /images/:key?ids=…&format=png&scale=2&use_absolute_bounds=true`, ≤40 ids per call, for every non-fill asset node plus one reference PNG per frame. **Download everything now** (fill URLs expire ≈14 days, render URLs ≈30). CDN downloads: ≤6 concurrent.
5. On 429 honour `Retry-After`, else back off 2, 4, 8… ≤60 s. After 3 consecutive 429s or 2 timeouts: follow instructions §8 (continue in screenshot mode from rendered images, flagged approximate, if renders still work; otherwise checkpoint and stop; always tell the user the retry window).
If your Figma tool only returns inline responses, fetch one top-level section at a time and distil it to the pruned fields immediately.

## 3. Normalise with code (keep only these fields)
`id, name, type, visible, box (relative to root), layoutMode/positioning/sizing/align, padding*, itemSpacing, layoutWrap, clipsContent, fills, strokes, strokeWeight(s), cornerRadius(es), effects, opacity, characters, text style (fontFamily, fontWeight, fontSize, lineHeightPx, letterSpacing, textCase, textDecoration, textAlign, hyperlink), runs/overrides, componentId, URL reactions, children`. Drop `visible:false` nodes, vector paths, prototype and plugin data. Pre-blend any alpha <1 solid colour against its parent background to a hex. Work from files; look at one section's slice at a time.

## 4. Map to the spec (`python3 "$EK" build --schema`)
| Figma | spec |
|---|---|
| first frame with logo, ≤160 px tall | `header` (`dark_src` if a dark logo exists) |
| headline + subtext + button (± image) | `hero`; text over an image → `hero.bg_image` (export the background **without** text) |
| stacked headings/paragraphs | `text` (mixed runs: `**bold**`, `[label](url)`; other styles → `raw`) |
| full-width image | `image` |
| horizontal layout of ≥2 cards that should stack | `columns` (compact rows such as icon+label or label/value stay in one row: `raw` via `build --snippet`, or a `table`) |
| text child + fill/stroke + radius/padding and a link, or name matches `/button\|btn\|cta/i` | `button` (never an asset) |
| label/value or price rows | `table` |
| ≤2 px line / empty frame | `divider` / `spacer {h}` |
| last section with unsubscribe/legal/address | `footer` |
| anything else | `raw` (use `build --snippet`) or flatten to an asset |

**Tokens** (derive in code across all text/fills): `page_bg`, `card_bg`, `text`, `muted`, `accent`, `link`, `rule`, `font_name`, `scale` (distinct text styles → h1, h2, h3, p, sub, legal), `btn` (radius, height, font size/weight, horizontal padding). Section-specific differences go in that section's `style` (`bg`, `pad`, `align`, per-role text overrides).
**Text roles:** use text-style names (H1, Heading 2…); else rank sizes. One `h1`.
**Links:** from hyperlinks/URL reactions. A button or link with no URL → `https://placeholder.invalid/<slug>-<node>` (blocking for send). Never invent URLs.
**Copy:** keep placeholder copy (lorem, "Headline goes here") as is and flag it. Derive the preheader from the headline + subhead (40–130 chars, word boundary); log it.
**Jinja:** dynamic only where text already contains `{{…}}`/`{%…%}` or the layer is named `#slot:<expr>||<fallback>`, `#loop:<var>`, `#if:<expr>`. Variable names come only from the design, the brief or memory. Fallback precedence: layer-name fallback, existing `default(...)`, brief, then the design sample (flag "review").
**Non-Auto-Layout frames:** order children by y then x, cluster into rows, derive padding/gaps from offsets, mark that section approximate.

## 5. Assets
- Collapse each vector-only subtree (icons, logos) into **one** asset at the outermost vector-only ancestor (render). Masks, rotation, blend modes, blur, complex gradients (>3 stops), stylised text → render and flatten. Image fills that are cropped/transformed/rounded or whose aspect differs from natural by >1% → render the node. Unmodified fills → fill. Text inside a flattened group: export only non-text layers and rebuild the text live; if impossible, flatten with `alt` = the text and flag.
- Optimise: JPEG (photos, q≈82–85) or PNG (graphics/transparency); GIF stays GIF; resize to ≤2× display width. Hash the optimised bytes; upload each unique image once to MoEngage image hosting; verify HTTPS + 200 + `image/*` and that it is not a Figma/S3 URL. Retry uploads 3×; if still failing, do not save dependents.
- Logo luminance: a transparent logo with mean opaque luminance <0.4 is dark-mode-at-risk; use a dark variant from Figma if one exists, otherwise flag (never invent one).

## 6. Fidelity (standard and full)
If the template tool returns a preview/thumbnail after saving, view it once and compare with the design reference. Otherwise, with Chromium ON:
After QA: `python3 "$EK" render out.html --sbs <reference.png> sbs.png`, view `sbs.png` once (reference left, render right). Patch the spec only for real differences (spacing, colour, missing element, wrong crop/aspect, wrong width); tolerate font-fallback wrapping and shadow/radius differences (log them). Full: crop only flagged sections (≤6). Stop at the loop limit and report residuals.
With no way to render (no Chromium, no platform preview), every spec value still traces to a node, so say "visual comparison not performed" in the report instead of claiming fidelity.
