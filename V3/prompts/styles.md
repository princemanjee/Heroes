# Genre / Tone / Language tables

These are the fixed choice lists. Ported verbatim from `V1/types.ts` (mirrored in `V3/_reference/types.ts`) so behavior matches the original app.

## Genres

1. Classic Horror
2. Superhero Action
3. Dark Sci-Fi
4. High Fantasy
5. Neon Noir Detective
6. Wasteland Apocalypse
7. Lighthearted Comedy
8. Teen Drama / Slice of Life
9. Custom *(user provides free-form premise — follow it strictly over genre tropes)*

### Genre → art-style string

Used verbatim as the `STYLE:` prefix in every panel prompt.

| Genre | Style string |
|---|---|
| Classic Horror | `Classic Horror comic book art, detailed ink, vibrant colors` |
| Superhero Action | `Superhero Action comic book art, detailed ink, vibrant colors` |
| Dark Sci-Fi | `Dark Sci-Fi comic book art, detailed ink, vibrant colors` |
| High Fantasy | `High Fantasy comic book art, detailed ink, vibrant colors` |
| Neon Noir Detective | `Neon Noir Detective comic book art, detailed ink, vibrant colors` |
| Wasteland Apocalypse | `Wasteland Apocalypse comic book art, detailed ink, vibrant colors` |
| Lighthearted Comedy | `Lighthearted Comedy comic book art, detailed ink, vibrant colors` |
| Teen Drama / Slice of Life | `Teen Drama / Slice of Life comic book art, detailed ink, vibrant colors` |
| Custom | `Modern American comic book art, detailed ink, vibrant colors` |

## Tones

Pick **one** tone per story. Some are genre-restricted (see next section).

- `ACTION-HEAVY` — Short, punchy dialogue. Focus on kinetics.
- `INNER-MONOLOGUE` — Heavy captions revealing thoughts.
- `QUIPPY` — Characters use humor as a defense mechanism.
- `OPERATIC` — Grand, dramatic declarations and high stakes.
- `CASUAL` — Natural dialogue, focus on relationships/gossip.
- `WHOLESOME` — Warm, gentle, optimistic.

### Genre → allowed tones

- **Teen Drama / Slice of Life**, **Lighthearted Comedy** → only `CASUAL`, `WHOLESOME`, `QUIPPY`
- **Classic Horror** → only `INNER-MONOLOGUE`, `OPERATIC`
- All other genres → any tone is fair game

If the user doesn't specify, pick one at random from the allowed set for their genre.

## Languages (for captions, dialogue, choices)

Scenes are always described in English (for the image model). Only the on-panel text is localized.

- `en-US` English (US)
- `ar-EG` Arabic (Egypt)
- `de-DE` German (Germany)
- `es-MX` Spanish (Mexico)
- `fr-FR` French (France)
- `hi-IN` Hindi (India)
- `id-ID` Indonesian (Indonesia)
- `it-IT` Italian (Italy)
- `ja-JP` Japanese (Japan)
- `ko-KR` Korean (South Korea)
- `pt-BR` Portuguese (Brazil)
- `ru-RU` Russian (Russia)
- `ua-UA` Ukrainian (Ukraine)
- `vi-VN` Vietnamese (Vietnam)
- `zh-CN` Chinese (China)

## Page schedule

- `MAX_STORY_PAGES = 10` — story beats
- `DECISION_PAGES = [3]` — page 3 presents the branching choice
- `BACK_COVER_PAGE = 11` — teaser, "NEXT ISSUE SOON"
- Cover = page 0

Narrative arc guidance by page number:

| Pages | Beat |
|---|---|
| 1 | Inciting incident. An event disrupts the status quo. Establish genre mood. |
| 2 | Rising action. Heroes engage with the new situation. |
| 3 | **Decision page.** End with a *psychological* choice about values / relationships / risk — not a physical "go left or right" choice. |
| 4 | Consequence of the choice begins to land. |
| 5–6 | Rising action, complicated by the branch the user took. |
| 7–8 | Complication. Secret revealed, misunderstanding deepens, or path blocked. |
| 9 | Build to climax. |
| 10 | **Final page.** Karmic cliffhanger. Must explicitly reference the page-3 choice and show how that philosophy led here. End with `TO BE CONTINUED...` (or localized equivalent). |

## Guardrails (negative constraints — always include)

1. UNLESS genre is Dark Sci-Fi, Superhero Action, or Custom: do not use jargon like "Quantum", "Timeline", "Portal", "Multiverse", or "Singularity".
2. IF genre is Teen Drama or Lighthearted Comedy: stakes must be social, emotional, or personal (rumors, competition, broken promises, embarrassment). Not life-or-death.
3. Avoid "The artifact" / "The device" phrasing unless already established in an earlier panel.
4. No repetition of previous captions or dialogue.
5. Vary shot composition — if the previous panel was action, make this one reaction or wide.
6. Don't say "CO-star" or "hero" literally in captions; use names (if established) or natural descriptors.
