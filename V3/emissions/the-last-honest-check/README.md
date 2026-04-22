# Emissions — The Last Honest Check in Bay City

Every Gemini-ready panel prompt emitted for this issue, saved to disk so terminal corruption can never lose one again.

## Usage

1. Open any `<page-code>.md` file in this folder.
2. `Ctrl+A`, `Ctrl+C` — the whole file is the pasteable block. No fences to strip, no commentary to skip.
3. Paste into Gemini AI Studio, attach the two reference images from `V3/pictures/`, submit.

## File naming

| File | Page | Beat |
|---|---|---|
| `00-cover.md` | 0 | Cover |
| `01.md` | 1 | Inciting incident — 2 AM office |
| `02.md` | 2 | Celeste arrives |
| `03.md` | 3 | The pitch (3-panel) |
| `04.md` | 4 | The finger-tap tell (3-panel) |
| `05.md` | 5 | Partner debrief (2-panel, height-contrast) |
| `06.md` | 6 | Pacific Building splash |
| `07.md` | 7 | Picking the lock (2-panel) |
| `08.md` | 8 | Penthouse split-location search (2-panel) |
| `09.md` | 9 | The laundering reveal (3-panel) |
| `10.md` | 10 | Midpoint shift — "we are the noose" (3-panel) |
| `11.md` | 11 | Hitman on the stairs (2-panel) |
| `12.md` | 12 | Laptop swing (4-panel action) |
| `13.md` | 13 | Prince takes the round (2-panel) |
| `14.md` | 14 | **DECISION POINT** — 500 feet (3-panel) |
| `15-B.md` | 15B | Sikorsky lands on Pier 19 |
| `16-B.md` | 16B | Alaric on the foredeck |
| `17-B.md` | 17B | Jamie walks the pier |
| `18-B.md` | 18B | The second gun appears |
| `19-B.md` | 19B | Jamie moves first (4-panel action) |
| `20-B.md` | 20B | Jamie down |
| `21-B.md` | 21B | Prince left-handed |
| `22-B.md` | 22B | Hospital reunion |
| `23-B.md` | 23B | Bedside warmth-through-pain |
| `24-B.md` | 24B | The check-taping ritual |
| `25-B.md` | 25B | **CLIFFHANGER** — TO BE CONTINUED |
| `26-back-cover.md` | 26 | Back cover teaser |

## Save status

Files currently saved: `23-B.md`, `24-B.md`, `25-B.md`, `26-back-cover.md`.

The earlier panels (00–22-B) were emitted to the terminal during the session and rendered without issue — but to protect against needing a reprint later, ask Claude to "save remaining emissions" and he'll write them in batches.

## Branch A

This issue ran branch B (principle path). Branch A (cash-the-check path) is fully specified in `V3/sessions/the-last-honest-check.json` under `arc.pages[15-25].branches.A` and can be emitted later with `/comic branch`. Those files will save here as `15-A.md` through `25-A.md`.
