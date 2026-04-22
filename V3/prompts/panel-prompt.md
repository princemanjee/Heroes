# Panel prompt template

This is the final output format — the block the user copy-pastes into Gemini AI Studio to render one panel. Emit one of these per page, gated behind the user saying "continue" after the previous one.

## Output structure

Every panel prompt is wrapped in a fenced code block so it copies cleanly. Structure:

```
=== PANEL {N} of 12 — {type} ===
ATTACH THESE REFERENCE IMAGES IN AI STUDIO:
  1. {hero_filename} (HERO)
  2. {sidekick_filename} (SIDEKICK)   ← only if sidekick exists

--- PROMPT START ---

STYLE: {style_string_from_styles_md}
ASPECT RATIO: {2:3 for story and cover, 2:3 for back cover}
TYPE: {Vertical comic panel | Comic book cover | Comic back cover}

CHARACTERS IN FRAME: {HERO | SIDEKICK | HERO and SIDEKICK | other}

CHARACTER LIKENESS (anchor to the attached reference images; textual description is a backup channel — if the attached images conflict with text, prefer the images for face/hair/build):

HERO:
{full hero character sheet from session JSON, as bullets}

SIDEKICK:                            ← only if sidekick exists
{full sidekick character sheet, as bullets}

WARDROBE THIS PANEL:
- HERO: {wardrobe.HERO from this page's arc entry}
- SIDEKICK: {wardrobe.SIDEKICK}       ← only if sidekick exists

SCENE: {scene string from arc, English, mentioning HERO / SIDEKICK tokens}

SHOT COMPOSITION: {optional — one-line direction like "low-angle hero shot, sidekick in background"; derived from the arc's beat and from rotating shot variety across panels}

TEXT ELEMENTS TO RENDER INSIDE THE PANEL:
- Caption box: "{caption in target language}"
- Speech bubble ({HERO|SIDEKICK} speaking): "{dialogue in target language}"

NEGATIVE CONSTRAINTS:
- Maintain strict facial likeness to the attached reference images.
- Do not drift from the wardrobe description above.
- Do not add extra text beyond the caption and speech bubble listed.
- Text inside the panel must appear exactly as written — do not translate, paraphrase, or drop it.

--- PROMPT END ---

REFERENCE IMAGE BASE64 (for API / inline-data consumers; ignore if you're using the web UI with attachments):

HERO_BASE64:
{base64 string, or the line "see sessions/refs/hero.base64" if too large to inline}

SIDEKICK_BASE64:                      ← only if sidekick exists
{base64 string, or "see sessions/refs/sidekick.base64"}
```

## Per-panel-type variations

### Cover (page 0)

- `TYPE: Comic book cover`
- `SCENE:` should be a dynamic hero action pose, cover-worthy.
- Add `TITLE TEXT:` line — `"INFINITE HEROES"` or the localized translation in the target language.
- Omit `Caption box` and `Speech bubble` fields.

### Story page (pages 1–10)

- `TYPE: Vertical comic panel`
- Include both caption and dialogue if the arc provides them. Omit whichever the arc leaves empty.
- For the decision page (page 3), **do not** render the choice options inside the panel. The choices are surfaced to the user in Claude Code's UI, not drawn into the image.

### Back cover (page 11)

- `TYPE: Comic back cover, full-page vertical art`
- `SCENE:` dramatic teaser, thematic.
- Add `TEASER TEXT:` — `"NEXT ISSUE SOON"` or localized.
- Omit caption/dialogue fields.

### Decision-page branches (after page 3)

Once the user picks A or B at page 3, only emit that branch's version for pages 4–10. The other branch stays in the arc JSON, available via `/comic branch` to explore later without overwriting.

## Inline base64 vs. sidecar files

Base64 for a 512×512 JPEG is roughly 40–60 KB. That's fine to inline once in conversation but awful to re-inline in all 12 panel prompts. Rule:

- **Small reference images (<50 KB encoded):** inline base64 in every prompt.
- **Larger references:** write base64 to `sessions/refs/<character>.base64` once, and in the prompt block say `HERO_BASE64: see sessions/refs/hero.base64`. The user can grep/cat that file when they need it.

This keeps the pasted prompt compact for AI Studio's text box while still giving API users something to splice in programmatically.

## Shot-composition rotation

Track the last two panels' shot type in session state (`lastShots: ["wide", "close-up"]`). When emitting a new panel, pick a composition that isn't in that list. Rough palette:

- Establishing wide
- Medium two-shot
- Medium single (hero or sidekick alone)
- Close-up reaction
- Close-up action
- Low angle (heroic)
- High angle (vulnerable)
- Over-the-shoulder
- Dutch angle (tension / unease)
- Silhouette / backlit

For the decision page (3) use a close-up reaction on the focus character. For the final page (10) use whichever composition maximizes the cliffhanger — often a pulled-back wide or a silhouette.

## Regeneration

If the user says "regenerate the page 5 prompt with more action" or similar, re-emit just that panel's block with the requested adjustment. Update the arc JSON's `scene` for that page if the user wants the change to stick; otherwise, treat it as a one-off variant. Overwrite the saved emission file too — it is the source of truth for that panel.

## Emission saving (non-negotiable)

Every emitted panel prompt is **saved to `V3/emissions/<slug>/<page-code>.md` before being printed to the terminal**. Terminal rendering of long prompts gets corrupted by CLI line-wrapping, copy-buffer quirks, and hook output; the saved file never does. The user should never have to ask for a reprint.

Filename convention:
- `00-cover.md` — cover
- `01.md` through `NN.md` — single-track story pages (NN is the last single-track page before the decision branches)
- `NN-A.md` / `NN-B.md` — post-decision branched pages (one file per branch per page)
- `NN-back-cover.md` — back cover (NN = last page number in the schedule)

The file content is the exact block that would be pasted to Gemini — no surrounding triple-backtick fences, no commentary, no header annotations. The file IS the pasteable content. Workflow: user opens the file, Ctrl+A, Ctrl+C, paste into AI Studio.

If the user says "reprint page X" or "give me panel X again", open the saved file and output its contents verbatim. Do NOT regenerate from scratch — that wastes tokens and can drift from the version the user may already be partway through rendering.

Session JSON must include `emissionsPath: "emissions/<slug>/"` so `/comic resume` can find them.
