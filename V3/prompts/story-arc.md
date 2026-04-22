# Story arc generation

V1 generated story beats one at a time, feeding each past panel's scene/caption/dialogue back into the next call. V3 does it differently: **write the whole 10-page arc up front, show it to the user, let them approve or edit, then emit panel prompts from the approved arc.**

This flips V1's model from "reactive" to "planned". The tradeoff — V1 could surprise itself mid-story; V3 stays coherent and lets you see where the story is going before you spend a single Gemini call.

## Inputs (from session state)

- Hero character sheet
- Sidekick character sheet (optional — if the user didn't provide one, invent a sidekick that fits the genre and generate a sheet for them too)
- Genre (from `styles.md`)
- Tone (from `styles.md`, respecting the genre → tone restrictions)
- Language (for captions/dialogue/choices; scenes stay in English)
- Custom premise (only if genre is Custom)
- Rich mode on/off (default on)

## Output format

A single JSON structure the user approves before we start emitting panel prompts. Structure:

```json
{
  "title": "[short punchy working title]",
  "logline": "[one-sentence premise]",
  "pages": [
    {
      "page": 0,
      "type": "cover",
      "focus_char": "hero",
      "scene": "Cover composition, English only — dynamic hero pose, cover-worthy angle",
      "title_text": "INFINITE HEROES (or localized)",
      "wardrobe": { "HERO": "[signature outfit for the issue]", "SIDEKICK": "[if present]" }
    },
    {
      "page": 1,
      "type": "story",
      "beat": "inciting_incident",
      "focus_char": "hero",
      "scene": "[English visual description — what we see. Must mention HERO and/or SIDEKICK.]",
      "caption": "[narrator text in target language — see limits below]",
      "dialogue": "[speech in target language — optional]",
      "wardrobe": { "HERO": "...", "SIDEKICK": "..." }
    },
    { "page": 2, "type": "story", "beat": "rising_action", "...": "..." },
    {
      "page": 3,
      "type": "decision",
      "beat": "values_choice",
      "focus_char": "hero",
      "scene": "...",
      "caption": "...",
      "dialogue": "...",
      "choices": [
        { "id": "A", "label": "[option A in target language]", "philosophy": "[one phrase — what this choice reveals about the hero]" },
        { "id": "B", "label": "[option B in target language]", "philosophy": "[one phrase]" }
      ],
      "wardrobe": { "HERO": "...", "SIDEKICK": "..." }
    },
    {
      "page": 4,
      "type": "story",
      "beat": "consequence",
      "branches": {
        "A": { "scene": "...", "caption": "...", "dialogue": "...", "focus_char": "...", "wardrobe": { } },
        "B": { "scene": "...", "caption": "...", "dialogue": "...", "focus_char": "...", "wardrobe": { } }
      }
    },
    "... pages 5-9 same branching structure ...",
    {
      "page": 10,
      "type": "story",
      "beat": "climax_cliffhanger",
      "branches": {
        "A": { "scene": "...", "caption": "[must reference page-3 choice A; ends with TO BE CONTINUED equivalent]", "dialogue": "...", "focus_char": "...", "wardrobe": { } },
        "B": { "scene": "...", "caption": "[must reference page-3 choice B; ends with TO BE CONTINUED equivalent]", "dialogue": "...", "focus_char": "...", "wardrobe": { } }
      }
    },
    {
      "page": 11,
      "type": "back_cover",
      "scene": "Dramatic teaser image, full-page vertical",
      "teaser_text": "NEXT ISSUE SOON (or localized)"
    }
  ]
}
```

## Writing rules (lifted from V1's `generateBeat`, adapted for whole-arc planning)

### Focus character rotation

The hero shouldn't be the camera's focus in every panel. Track `focus_char` across pages and rotate: if pages 1 and 2 both focused on the hero, page 3 must focus on the sidekick (or on something else relevant — an antagonist, an object). Aim for roughly 50/50 hero/sidekick share across the 10 story pages when a sidekick exists.

### Shot variety

Don't emit three consecutive close-ups or three consecutive wide shots. Think like a storyboard artist: establishing → medium → close → reaction → wide → action. Vary it.

### Wardrobe continuity with intentional changes

Wardrobe is part of the story, not just set dressing. Keep it consistent within a scene, but let it change meaningfully when the story demands:

- Cover and page 1 usually match — the "signature look" for the issue.
- A wardrobe change across pages signals a scene break, passage of time, or story shift (e.g. changing into combat gear, getting soaked in the rain, dressing up for a formal event).
- Track accumulated damage — if page 5's jacket got torn, pages 6+ show the tear until a narrative reason resets it.

Record wardrobe **per page per character** in the arc JSON so panel prompts can reference it deterministically.

### Language split

- `scene` always English — it feeds the image model.
- `caption`, `dialogue`, `choices[].label` in the target language.
- Character descriptors inside scenes use the canonical English labels `HERO` and `SIDEKICK` (or the names if the user named them) — never translate those tokens.

### Decision page (page 3)

- The two choices must be *psychological* — about values, relationships, or risk tolerance — not physical actions.
- Good examples: "Tell her the truth / Protect her from it", "Keep the promise / Save the stranger", "Trust the mentor / Trust your gut".
- Bad examples: "Go left / Go right", "Fight / Run".
- Each choice gets a one-phrase `philosophy` label. Page 10 will call back to whichever label was chosen.

### Final page (page 10)

- Must explicitly reference the page-3 choice in the caption — show how that philosophy led here.
- Must end with the target-language equivalent of `TO BE CONTINUED...`.
- Both branches (A and B) should land on *different* cliffhangers, not the same ending dressed up.

### Text length — rich mode on

- Caption: max 35 words. Detailed narration or internal monologue.
- Dialogue: max 30 words. Rich, character-driven speech.

### Text length — rich mode off

- Caption: max 15 words.
- Dialogue: max 12 words.

### Guardrails

See `styles.md` § Guardrails. Include a mental check against each one while writing.

## Approval loop

After drafting, present the arc to the user in a readable summary (not the full JSON — Markdown page-by-page bullets). Ask specifically:

1. Does the logline match what they wanted?
2. Are the two page-3 choices *meaningfully different* in philosophy?
3. Does the final page land a cliffhanger that earns a sequel?
4. Any page they want rewritten before we start emitting Gemini prompts?

Only once the user says "go" do you save the arc to the session JSON and move to panel-prompt emission.
