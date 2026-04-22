# V3 — Infinite Heroes (Claude Code workflow)

V1 was the original React app that called Gemini directly for both text and images. V2 was a refactor that dropped image generation and came out worse. **V3** replaces the React app entirely with a Claude Code workflow: Claude writes the story and emits paste-ready Gemini image prompts, one panel at a time. You run the prompts through the Gemini web UI (AI Studio) to actually render the panels.

## What this is

- **Input:** reference images of a Hero and a Sidekick (face, hair, build, gait — the things that need to stay consistent across panels).
- **Output:** a full 10-page comic story plus 12 Gemini-ready image prompts (cover + 10 story pages + back cover), each one including the character descriptions and scene-specific wardrobe so Gemini can keep likeness while letting costume change with the story.
- **Branching:** page 3 is a decision page. You pick one of two choices; the workflow writes pages 4–10 from that branch. You can go back later and explore the alternate branch without losing the original.

## How to run it

1. Drop your hero and sidekick reference images anywhere under this folder (or keep them wherever — you'll just give Claude the paths).
2. In Claude Code, from the `V3/` directory, run:

   ```
   /comic
   ```

3. Claude will walk you through:
   - Hero + sidekick image paths → extracts detailed character sheets
   - Genre, language, optional custom premise, tone
   - 10-page story arc (approved by you before any prompts are emitted)
   - Cover prompt → you paste into Gemini → you come back
   - Pages 1, 2, 3 (decision) — you pick a choice
   - Pages 4–10 built from the choice
   - Back cover
4. Your story state is saved to `sessions/<slug>.json`. Reopen any session with `/comic resume <slug>`.

## Feeding prompts into Gemini

Each panel prompt is a clearly delimited block you copy-paste into Gemini (AI Studio web UI, `gemini-3-pro-image-preview` or newer). Every block contains:

- **Reference images to attach** — both hero and sidekick, attached to every panel so likeness holds even when they're not in frame (the model still anchors style).
- **Base64 blob of each reference image** — embedded in the prompt as a backup/secondary channel if you're running somewhere that accepts inline data.
- **Character likeness notes** — detailed textual description of face/hair/build/posture derived once from the uploaded photos.
- **Wardrobe for this panel** — scene-appropriate costume (varies with story beats).
- **Scene, caption, dialogue** — the visual direction plus any text that should render inside the panel.

## Directory layout

- `.claude/commands/comic.md` — the `/comic` slash command. This is the workflow Claude executes.
- `prompts/` — reusable prompt templates Claude composes into the final Gemini prompts.
- `sessions/` — per-story state (JSON). Gitignored except for `example.json`.
- `_reference/` — a copy of V1's `App.tsx` and `types.ts`. The prompt logic in V3 was ported from there; keep the copies so the prompt rules stay traceable to the working V1 implementation.

## Why not just use V1?

V1 works end-to-end but locks you into what Gemini returns on the first try. V3 lets you:
- See and edit the full story arc before any image is generated.
- Regenerate any panel's prompt with tweaks before spending a Gemini call.
- Branch on the decision page and keep both branches on disk.
- Use whatever Gemini surface you prefer (AI Studio, API, batch).
