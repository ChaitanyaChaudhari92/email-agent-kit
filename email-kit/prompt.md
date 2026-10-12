# Prompt mode (text only)

1. **Parse the brief** into: email type (`promotional | lifecycle | newsletter | transactional | alert`), goal, audience, tone, primary CTA (label + URL), key messages, **verbatim facts** (offers, dates, prices, codes, legal), personalization fields (expression + fallback), language. Fill missing non-essentials with defaults and log them. Missing URLs → `https://placeholder.invalid/<slug>` (blocking for send). Facts not supplied are **never invented**: use `[PLACEHOLDER: …]`.
2. **Brand tokens (in order):** brief → memory `design_tokens` → tokens extracted from the 3 most recent templates of this brand in the same store (search; parse colours, font stack, button style, width) → builder defaults (flag "default brand tokens, replace"). Reuse the registry header/footer block if one exists. Never invent a logo: registry block, an existing hosted logo, or a placeholder slot.
3. **Pick the recipe** and do not add sections the brief did not imply:

| Type | Sections |
|---|---|
| promotional | header · hero (headline, subhead, CTA, optional image) · text · optional 2-up columns · optional offer line (only from verbatim facts) · optional CTA band · footer |
| lifecycle | header · text-led hero · 3 benefit/step rows (`columns` or `text`) · CTA · support line · footer |
| newsletter | header · intro `text` · article `columns`/rows (≤5: image, h3, excerpt, link) · footer |
| transactional | header · h1 status (`hero` without button) · `table` of details · optional CTA · help line · footer (no marketing unsubscribe unless asked) |
| alert (Inform) | minimal header · h1 · message `text` · optional `table` · optional CTA · minimal footer |

4. **Write the copy** from key messages and verbatim facts only: one idea per section; headline ≤8 words; CTA 2–4 words; no invented urgency/scarcity; no spam-trigger styling (ALL CAPS shouting, "!!!"). Write in the brief's language; never translate. Mark all generated copy for review in the report. Derive the preheader (40–130 chars) from the copy.
5. **Images:** each image slot gets its recipe size (hero 600×300, card 260×160, logo 160×40). Create it with `python3 "$EK" placeholder <w> <h> <name>.png` (flat `#D9DEE3`, 2×), host it, `placeholder-…` name, list as blocking for send; alt text from the slot's purpose. Never use AI-generated or stock images unless the user supplied them.
6. **Spec → build → QA** per instructions §5. Review against the brief once (no reference image exists): one clear primary CTA visible on mobile without scrolling much, headline → support → CTA order, every key message present, nothing claimed that the brief did not state.
