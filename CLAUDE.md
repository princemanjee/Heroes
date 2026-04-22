# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository layout

Three sibling top-level folders, each a different take on **Infinite Heroes** — a branching, localized comic-book generator.

- **`V1/`** — original Vite + React 19 + TypeScript app. Calls Gemini directly for both text and images. Working end-to-end. Do not modify without explicit ask.
- **`V2/`** — an earlier refactor of V1. Treated as archived/intact; do not modify. Differs from V1 in model split, aspect ratio, and failure UI (see below).
- **`V3/`** — **not** a React app. A Claude Code workflow: Claude writes the story and emits paste-ready Gemini image prompts, user drives Gemini manually via the web UI. This is the active surface for new work.

### V1 vs V2 (both intact)

The two React folders are near-duplicates. Known differences:

- **Models** — V1 uses `gemini-3-pro-image-preview` for both text and image calls. V2 splits them: `gemini-3-pro-image-preview` for images, `gemini-3-flash-preview` for text.
- **Image aspect ratio** — V1 requests `2:3`, V2 requests `3:4`.
- **Image-failure UI** — V2's `Panel.tsx` renders an "Art Block!" fallback when `imageUrl === 'FAILED'`; V1 silently shows nothing. V2's `App.tsx` writes the `'FAILED'` sentinel, V1 leaves `imageUrl` empty.
- **Scripts** — V2's `package.json` adds `"lint": "tsc --noEmit"`; V1 has no lint script.

### V3 — Claude Code workflow

V3 has no build, no package.json, no dev server. Its "runtime" is Claude Code executing the `/comic` slash command.

- `V3/.claude/commands/comic.md` — the `/comic` slash command Claude follows (orchestration spine). This file defers to the `prompts/` files for detail.
- `V3/prompts/styles.md` — genres, tones, languages, page schedule, guardrails (ported from `V1/types.ts` and `V1/App.tsx`'s `generateBeat` prompt).
- `V3/prompts/character-sheet.md` — extract structured face/hair/build from a reference image.
- `V3/prompts/story-arc.md` — write the whole 10-page arc (both branches for pages 4–10) before emitting any prompts. This is the key V3-vs-V1 shift: V1 was reactive per-beat, V3 is plan-first so the user can approve before spending Gemini calls.
- `V3/prompts/panel-prompt.md` — the exact output format Claude emits per panel for paste into Gemini AI Studio. Includes attach-image directives, textual character-likeness, per-panel wardrobe, base64 sidecar handling.
- `V3/sessions/` — per-story JSON state (gitignored except `example.json`). `V3/sessions/refs/` holds base64 of user-uploaded reference images.
- `V3/_reference/` — verbatim copies of `V1/App.tsx` and `V1/types.ts`. V3's prompt rules were ported from there; keep the copies so drift between V1 behavior and V3 prompts stays auditable.

When editing V3: the rule in `comic.md` is that if its orchestration text conflicts with the specific `prompts/*.md` files, the specific file wins. Update both sides if you change shared rules.

## Commands

### V1 / V2 (React apps)

Run from inside `V1/` or `V2/`:

- `npm install` — install deps
- `npm run dev` — Vite dev server on port 3000, host `0.0.0.0`
- `npm run build` — production build
- `npm run preview` — serve built output
- `npm run lint` — **V2 only** — `tsc --noEmit` type-check. There is no ESLint/Prettier/test runner configured.

Set `GEMINI_API_KEY` in the folder's `.env.local`. `vite.config.ts` injects it as both `process.env.API_KEY` and `process.env.GEMINI_API_KEY` at build time — the runtime code reads `process.env.API_KEY`.

### V3 (Claude Code workflow)

No build commands. Invoke the workflow from Claude Code while in the `V3/` directory:

- `/comic` — start a new story
- `/comic resume <slug>` — resume a saved session
- `/comic list` — list saved sessions
- `/comic branch` — after page 3 is decided, emit the opposite branch's pages 4–10

Sessions land in `V3/sessions/<slug>.json`. Reference image base64 lands in `V3/sessions/refs/<slug>-<char>.base64`.

## Architecture

### Generation pipeline (App.tsx)

`App.tsx` is the orchestrator and holds all Gemini calls. The flow per issue:

1. **`launchStory`** — picks a random tone filtered by genre, seeds a `cover` face, then kicks off `generateBatch(1, INITIAL_PAGES)` and `generateBatch(3, 3)` in the background.
2. **`generateBatch(startPage, count)`** — reserves page numbers in a `generatingPages` `Set<number>` ref so concurrent batches do not duplicate work, appends placeholder `ComicFace` entries, then loops sequentially calling `generateSinglePage`.
3. **`generateSinglePage`** — routes by `type`:
   - `cover` / `back_cover` → skip beat generation, go straight to image.
   - `story` → call `generateBeat` → if the returned `focus_char === 'friend'` and no friend is uploaded, call `generatePersona` to synthesize a sidekick reference image on the fly → call `generateImage`.
4. **`generateBeat`** — text model, JSON-only response (`responseMimeType: 'application/json'`). Post-parse cleanup strips `"` around dialogue, drops `choices` on non-decision pages, and coerces `focus_char` to a valid enum. On failure returns a generic stub so the UI keeps moving.
5. **`generateImage`** — image model, multi-part `contents` array: hero base64 → friend base64 → prompt text. Style string is derived from the genre (or `"Modern American"` for Custom).
6. **`handleChoice`** — after the decision page is resolved, triggers `generateBatch(maxPage + 1, BATCH_SIZE)`.

### State model: why `useRef` everywhere

Async AI calls span many renders. The app keeps *two* copies of hero/friend/history state:

- `useState` for render (`hero`, `friend`, `comicFaces`).
- `useRef` mirrors (`heroRef`, `friendRef`, `historyRef`) that AI callbacks read.

`setHero` / `setFriend` write both. `updateFaceState` writes the ComicFace into both the state array *and* `historyRef.current` at the matching index. When editing state flow, update both sides — reading `hero` inside an async closure will give stale data.

### Prompt structure (critical to preserve when refactoring)

The beat prompt in `generateBeat` has several interlocking pieces; breaking one usually breaks the story:

- **`coreDriver`** — genre+tone normally, but replaced wholesale for `Custom` so the model follows the user's premise over genre tropes.
- **`guardrails`** — negative constraints preventing every story from drifting into "quantum multiverse" and keeping Teen Drama / Comedy stakes social rather than life-or-death.
- **Co-star injection** — if a friend exists, ~60% of the time a "MANDATORY: FOCUS ON THE CO-STAR" line is appended to rebalance screen time against the last panel's `focus_char`.
- **Narrative-arc instructions** — page-number-gated (inciting incident → rising → complication → climax) with the decision page (page 3) and final page (page 10) as special cases.
- **Language split** — user-facing `caption`/`dialogue`/`choices` must be in the selected `LANGUAGES` entry, but `scene` stays in English because that string is fed to the image model.

When editing prompts: keep `scene` English-only, keep the decision page tied to `DECISION_PAGES` in `types.ts`, and keep the "OUTPUT STRICT JSON ONLY" closing block — the response parser strips ```json fences but assumes valid JSON.

### Page constants (types.ts)

`MAX_STORY_PAGES = 10`, `BACK_COVER_PAGE = 11`, `TOTAL_PAGES = 11`, `DECISION_PAGES = [3]`, `INITIAL_PAGES = 2`, `BATCH_SIZE = 6`. Changing these is safe in isolation, but the beat prompt has hardcoded references to page 3 (the decision page) and "FINAL PAGE" logic keyed off `MAX_STORY_PAGES` — audit both if you shift the schedule.

### API key handling

`useApiKey.ts` integrates with AI Studio's injected `window.aistudio` object (`hasSelectedApiKey` / `openSelectKey`). In a plain browser there is no `window.aistudio`, so `validateApiKey` returns `true` and the app falls back to `process.env.API_KEY` baked in by Vite. `handleAPIError` inspects error messages for `API_KEY_INVALID` / `permission denied` / `Requested entity was not found` and re-opens the key dialog — preserve this string-match list when touching error handling.

### UI components

- **`Setup.tsx`** — pre-flight form (hero/friend upload, genre/language/premise/richMode). Shown until `launchStory` runs.
- **`Book.tsx`** — paginated reader, shows `comicFaces` as a bound book.
- **`Panel.tsx`** — individual page; V2 handles the `'FAILED'` sentinel, V1 does not.
- **`LoadingFX.tsx`** — loading animation for in-flight pages.

## Styling

Tailwind is loaded via CDN in `index.html` (no build-time Tailwind). `index.css` is empty. Fonts (`Bangers`, `Comic Neue`) are pulled from Google Fonts in `index.html`. There is no CSS module or component library.

## Gotchas

- `process.env.API_KEY` is substituted at build time by `vite.config.ts` — changing `.env.local` requires restarting `npm run dev`.
- The `.env.local` files in both folders are committed but effectively empty (35 bytes). Do not commit real keys.
- `jspdf` is used in `downloadPDF` and assumes base64 `data:` URLs in `face.imageUrl` — V2's `'FAILED'` sentinel would crash `addImage`; the current code filters on `face.imageUrl && !face.isLoading` but not on the sentinel. If you change the failure representation, update the PDF filter too.
- All Gemini calls go through a fresh `new GoogleGenAI({ apiKey: process.env.API_KEY })` instance via `getAI()`. Do not hoist this to a module-level singleton without re-checking the AI Studio key-rotation flow.
