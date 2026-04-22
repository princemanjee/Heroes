# Character sheet extraction

Run this once per character (hero, sidekick) right after the user supplies a reference image. The output gets stored in the session JSON and inlined into **every** panel prompt — it's the persistent likeness anchor across all 12 panels.

## Input

A single reference photo (file path from user). Read it with the Read tool (Claude's vision handles the image inline).

## What to extract

Produce a structured description capturing only things that stay constant across costume changes and scenes. Ignore the clothing in the reference photo — costume is decided per-panel by the story, not by the source image.

Required fields:

- **Identity tag** — short label used in prompts (`HERO`, `SIDEKICK`, or the character's name if the user names them).
- **Apparent age range** — e.g. "late 20s", "teens".
- **Build** — height impression, frame (slight / athletic / broad / heavy), posture.
- **Face shape** — oval / round / square / heart / long, plus jawline and cheekbone notes.
- **Skin tone** — describe with a neutral, consistent vocabulary (e.g. "warm medium-brown", "fair with pink undertones"). Avoid food metaphors.
- **Eyes** — color, shape (almond / round / hooded / monolid), spacing, brow shape.
- **Nose** — size, bridge, tip.
- **Mouth** — lip fullness, resting expression.
- **Hair** — color, length, texture (straight / wavy / curly / coily), style in the reference, hairline.
- **Distinguishing marks** — freckles, scars, moles, tattoos, piercings, facial hair.
- **Gait / carriage** — only if inferable from the photo (posture, stance). Skip if not visible.
- **Default neutral expression** — what their face looks like at rest.

## Output format

Emit one Markdown block per character, structured exactly like this so the panel-prompt template can slot it in verbatim:

```
CHARACTER: HERO
- Age: [apparent age range]
- Build: [frame + posture]
- Face: [shape, jaw, cheekbones]
- Skin: [tone]
- Eyes: [color, shape, spacing, brows]
- Nose: [size, bridge, tip]
- Mouth: [lip fullness, resting expression]
- Hair: [color, length, texture, style, hairline]
- Marks: [freckles / scars / tattoos / piercings / facial hair — or "none visible"]
- Carriage: [posture notes — or "neutral"]
- Neutral expression: [what their face does at rest]
```

## Rules

1. **Do not invent features** that aren't in the photo. If hair is tied back, don't guess the untied length — say "tied back; untied length not visible".
2. **Be specific but brief** — each bullet should be one short phrase, not a sentence. Gemini reads these every panel; bloat adds up.
3. **Describe the person, not the photo** — no "wearing a blue shirt", no "lit from the left", no "against a white wall". Those belong in panel wardrobe / scene, not the character sheet.
4. **Use the same vocabulary across both characters** so comparisons (tall vs. short, darker vs. lighter skin) are consistent.
5. **Ask the user to confirm** the sheet before proceeding — misreading the reference image on the first panel cascades into every panel.

## After confirmation

Save the sheet into the session JSON under `characters.hero.sheet` (or `sidekick.sheet`). Also base64-encode the reference image and save under `characters.hero.base64` (or a sidecar file under `sessions/refs/` if the base64 is large). The panel-prompt template will pull both.
