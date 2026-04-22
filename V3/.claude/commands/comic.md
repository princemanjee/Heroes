# /comic — Infinite Heroes workflow

You are running the V3 comic generation workflow. Your job: turn user-supplied hero + sidekick reference images into a full 10-page branching comic, emitted as paste-ready Gemini image prompts.

## Arguments

- `/comic` — start a new story (default)
- `/comic resume <slug>` — resume `sessions/<slug>.json`
- `/comic list` — list saved sessions
- `/comic branch` — at/after page 3, emit the other branch you didn't take

## Source-of-truth files

Read these at the start of every run (they define the rules you follow):

- `V3/prompts/styles.md` — genres, tones, languages, page schedule, guardrails
- `V3/prompts/character-sheet.md` — how to extract and format character descriptions from a reference image
- `V3/prompts/story-arc.md` — how to write the 10-page arc JSON
- `V3/prompts/panel-prompt.md` — the exact output format for each Gemini-ready panel prompt

If any of those four files conflict with what's written below, **the specific prompt file wins** — those hold the detailed rules; this file is the orchestration spine.

## Workflow

### Step 0 — Parse arguments

If `resume <slug>`: load `V3/sessions/<slug>.json`, tell the user what page they're on, jump to Step 6 (panel emission) from that page.

If `list`: `ls V3/sessions/*.json`, print each slug + title + current page. Done.

If `branch`: requires an active session where page 3 is past. Switch the session's `activeBranch` to the other choice and resume panel emission from page 4. Done.

Otherwise (new story): continue to Step 1.

### Step 1 — Reference images

Ask the user for two file paths:

1. Hero reference image (required)
2. Sidekick reference image (optional — they can type "skip" and you invent one later)

Read each image with the Read tool (Claude's vision handles it inline). If the path doesn't exist, say so and re-ask.

### Step 2 — Character sheets

For each image, follow `prompts/character-sheet.md` to produce the structured description. Show both sheets to the user and ask them to confirm or edit before moving on. Misreads here poison every panel.

If no sidekick image was supplied, skip this step for the sidekick — you'll generate one in Step 4 once the genre is chosen.

### Step 3 — Story setup

Ask, in one message (not four back-and-forth questions):

1. **Genre** — list all 9 from `styles.md`, let them pick by number or name.
2. **Language** — for captions/dialogue/choices. Default `en-US`.
3. **Custom premise** — only if they picked Custom as the genre.
4. **Rich mode** — on (default) = longer captions/dialogue; off = terse.
5. **Tone** — or "auto" to pick from the genre's allowed list.

### Step 4 — Synthesize missing sidekick (if needed)

If the user skipped the sidekick image, invent a sidekick that fits the genre and run the character-sheet template on the invented character. Describe their face/hair/build in the same format. Show it to the user, ask "keep or regenerate?". The synthetic sidekick will not have a reference image to attach in Gemini — panel prompts for them will rely on the textual description only, and should say so explicitly in the "attach" line (e.g. `SIDEKICK — no reference image, rely on textual description below`).

### Step 5 — Story arc

Follow `prompts/story-arc.md`. Produce the full 10-page arc JSON (both branches for pages 4–10), then summarize it back to the user as readable page-by-page bullets. Ask the approval questions listed at the bottom of that file. Iterate until they say "go".

Save to `V3/sessions/<slug>.json`. Slug = lowercased-kebab of the title, e.g. `rooftop-bargain.json`. If the file already exists, append `-2`, `-3`, etc.

Session JSON structure:

```json
{
  "slug": "...",
  "createdAt": "ISO timestamp",
  "updatedAt": "ISO timestamp",
  "genre": "...",
  "tone": "...",
  "language": "en-US",
  "customPremise": "..." ,
  "richMode": true,
  "characters": {
    "hero": {
      "imagePath": "...",
      "base64File": "sessions/refs/<slug>-hero.base64",
      "sheet": "... full character sheet as markdown bullets ..."
    },
    "sidekick": {
      "imagePath": "...",
      "base64File": "sessions/refs/<slug>-sidekick.base64",
      "sheet": "...",
      "synthetic": false
    }
  },
  "arc": { ... full JSON from prompts/story-arc.md ... },
  "progress": {
    "currentPage": 0,
    "activeBranch": null,
    "lastShots": [],
    "emitted": []
  }
}
```

Also write base64 of each reference image to `sessions/refs/<slug>-<char>.base64` so panel prompts can reference the file by path instead of re-pasting the blob.

### Step 6 — Panel emission loop

This is the main interactive loop. For each page in order (0 = cover, 1, 2, 3, 4…10, 11 = back cover):

1. Compose the panel prompt using `prompts/panel-prompt.md`, pulling:
   - Style string from genre → `styles.md`
   - Character sheets from session JSON
   - Page's scene / caption / dialogue / wardrobe from the arc
   - Shot composition chosen so it differs from the last two panels (`progress.lastShots`)
2. **Save the prompt to disk before emitting.** Write the complete prompt block to `V3/emissions/<slug>/<page-code>.md`. Create the directory tree if missing. Filename convention: `00-cover.md`, `01.md` through `14.md` for single-track pages, `15-A.md` / `15-B.md` through `25-A.md` / `25-B.md` for branched pages, `26-back-cover.md` (or equivalent if the page schedule differs). The saved file is the canonical pasteable copy — terminal rendering of long prompts can be corrupted by CLI wrapping, copy-buffer issues, or hook output; the file never is.
3. Emit the block to the terminal exactly as specified in `prompts/panel-prompt.md`, identical to what was saved in step 2.
4. Update `progress.currentPage`, push the chosen composition onto `progress.lastShots` (keep last 2), append this page to `progress.emitted`, bump `updatedAt`, save the session JSON.
5. **Stop and wait for the user.** They'll either:
   - Say "continue" / "next" → go to the next page.
   - Paste back the rendered image or a description → acknowledge, save a note on the session, go to the next page.
   - Say "regenerate" with a tweak → re-emit just this panel with the tweak; update the arc's scene field if they want the change to stick.
   - At page 3 only: pick choice A or B → set `progress.activeBranch` and proceed into that branch's pages 4–10.
   - Say "pause" / "quit" → confirm saved, stop cleanly.

### Step 7 — Decision page behavior

At page 3, after emitting the prompt, surface both choices to the user as a numbered list with their philosophy labels. Wait for them to pick. **Do not** render the choices inside the image — they live in Claude's response only.

After the user picks, write `progress.activeBranch = "A"` (or `"B"`) and continue into the branch's pages 4–10. The unused branch stays in `arc.pages[n].branches.<other>` — it's not deleted.

### Step 8 — After page 11 (back cover)

Tell the user the issue is complete. Summarize:

- Title
- Total panels emitted
- Branch taken (A or B, + philosophy label)
- Path to the session JSON
- Reminder that `/comic branch` will emit the other branch's pages 4–10 from the same arc

## Hard rules

1. **Never emit a panel prompt until the user has approved the story arc in Step 5.** Arc first, prompts second.
2. **One panel at a time.** Never batch-emit multiple panels unless the user explicitly asks for "all remaining panels" (and even then, warn that they lose the regenerate-mid-story affordance).
3. **Save after every panel.** The session JSON should always reflect the latest state so a crashed session can resume.
4. **Character sheets are immutable once approved.** Wardrobe changes per panel, but face / hair / build / marks stay identical across all 12 prompts. If the user wants to change the character sheet mid-story, offer to restart the story (the likeness anchor changes everything).
5. **Scenes stay English; captions/dialogue/choices use the target language.** Character tokens (`HERO`, `SIDEKICK`) are never translated — they're labels the image model reads, not narrative text.
6. **Respect the guardrails in `styles.md`.** Especially: no quantum-multiverse jargon unless the genre calls for it, and social/emotional stakes for Comedy and Teen Drama.
7. **No code generation.** V3 emits prompts, not code. Do not offer to "just write a script that…" — the whole point of V3 is that the user drives Gemini manually.
8. **Every emission is saved to disk before it is emitted to the terminal.** `V3/emissions/<slug>/<page-code>.md` is the canonical pasteable copy. If the user asks to reprint a page, open the saved file and output its contents verbatim — do not regenerate from scratch. Set `emissionsPath` on the session JSON so resumed sessions know where the files live.
