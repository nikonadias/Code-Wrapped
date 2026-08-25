---
name: code-wrapped
description: Generate a shareable "Code Wrapped" stats card (Spotify-Wrapped style) from a git repo's history — lines written, commits, files, focused hours, a 24-hour coding clock, a docs-vs-code split, and an era time-machine (how long the same work would have taken in 2000/2010/2020/Nov 2025 vs now). Use when the user asks for a "code wrapped", "coding wrapped", "git wrapped", a coding-stats card, or a year/sprint-in-review graphic for a repository.
---

# Code Wrapped

Produces a self-contained, Instagram-shareable HTML card (navy `#0b1c3f`, off-white `#f4f3ee`, orange `#FF5B1F`, black `#0a0a0a`) at 1080px wide. Re-theme by editing the `:root` block in `template.html`. The card sizes to its own content (roughly 9:16) and renders on a seamless navy body — no scrolling, no dead space.

## Inputs
- **repo** — absolute path to a git repo. Default: current working directory.
- **name** — the eyebrow over the title (e.g. `NIKO`). Ask if unknown.
- **visibility** — the card is a local HTML file. It is never published anywhere; the user screenshots and shares it themselves.
- **range** — optional date window. Default: full history. To scope, append `--since=<date> --until=<date>` to every git command below.
- **out** — output path. Default: `<repo>/scratch/code-wrapped/index.html`.

## Step 1 — gather raw stats (run these in `<repo>`; they're the verified commands)

Use the Bash tool. All use `--all --no-merges` for the full picture.

1. **Commits + span:**
   `git log --all --no-merges --date=iso --pretty=format:"%ad" | sort` → first/last + `git rev-list --all --no-merges --count`.

2. **Lines written** (cleaned of lockfiles / vendored / db / generated):
   ```
   git log --all --no-merges --numstat --pretty=format: | awk '
   NF==3 && $1!="-"{p=$3; if (p ~ /package-lock\.json|node_modules\/|\.lock$|\.db$|\.min\.|dist\/|vendor\//) next; add+=$1} END{print add}'
   ```
   (extend the regex to skip any project-specific vendored dirs).

3. **Files touched (unique):**
   `git log --all --no-merges --name-only --pretty=format: | awk 'NF' | sort -u | wc -l`

4. **Focused hours** (session-gap method; a gap > 90 min starts a new session, sum the in-session gaps — this is a conservative floor):
   ```
   git log --all --no-merges --pretty=format:"%at" | sort -n | awk '
   {t=$1; if(prev){d=t-prev; if(d<=5400) total+=d} prev=t} END{printf "%.0f\n", total/3600}'
   ```

5. **Hour-of-day distribution** (the 24-int clock array, local time):
   ```
   git log --all --no-merges --date=format-local:'%H' --pretty=format:"%ad" | sort | uniq -c
   ```
   Build a 24-element array indexed 0..23 (fill missing hours with 0).

6. **Docs vs code** (commit-type prefixes):
   ```
   git log --all --no-merges --pretty=format:"%s" | sed -E 's/^([a-zA-Z]+).*/\1/' | tr 'A-Z' 'a-z' | sort | uniq -c
   ```
   `docs` = the `docs` count. `code` = everything else with a conventional prefix (feat, fix, refactor, perf, test, style, chore, build, wip…). Compute `CODE_PCT` and `DOCS_PCT` so they sum to 100 (round).

## Step 2 — derive the display values

- **LINES / LINES_SUFFIX**: round lines to a clean unit. `302341 → 302 / K`; `1.2M → 1.2 / M`. Keep it short.
- **FILES / COMMITS / HOURS**: thousands-separated integers (e.g. `3,010`).
- **CLOCK_DATA**: the 24-int array as a JS literal, e.g. `[28,45,42,...]`.

### Era time-machine (the bottom grid)
Answer: *"at 20 hours of code a day (a 2-person team), how long would `LINES` take with each era's tools?"* Use net delivered lines-per-person-hour (LPH) that climbs with tooling, against 20 person-hours/day, every day:

| Era | tooling label | LPH | formula |
|-----|---------------|-----|---------|
| 2000 | by hand | 4 | days = LINES / (4 × 20) |
| 2010 | forums | 10 | days = LINES / (10 × 20) |
| 2020 | frameworks | 21 | days = LINES / (21 × 20) |
| Nov 2025 | AI agents | 83 | days = LINES / (83 × 20) |
| This week | Claude | — | the **actual** elapsed time of the range (the span from Step 1) |

Convert days → the cleanest human unit and **round to a clean number** (years if ≥ ~400 days, months if ≥ ~50 days, else weeks). For ~300K lines the canonical set is **10 years · 4 years · 2 years · 6 months · 6 weeks**. Fill `ERA_2000`/`ERA_2000_U` … `ERA_NOW`/`ERA_NOW_U` (value + unit like `years`/`months`/`weeks`).

- **KICKER_A / KICKER_B**: orange lead + muted tail. Default: `More progress in 6 months` / `than the 20 years before it.` Adjust the "6 months" to match the Nov 2025 → now gap.

## Step 3 — render

Copy `template.html` (next to this file) to `out`, substituting every `{{PLACEHOLDER}}`. Create the output dir if needed. Placeholders: `NAME, LINES, LINES_SUFFIX, FILES, COMMITS, HOURS, CODE_PCT, DOCS_PCT, CLOCK_DATA, ERA_2000, ERA_2000_U, ERA_2010, ERA_2010_U, ERA_2020, ERA_2020_U, ERA_NOV, ERA_NOV_U, ERA_NOW, ERA_NOW_U, KICKER_A, KICKER_B`.

## Step 4 — verify alignment (do not skip)

Render it in a real browser and confirm the layout **numerically** before declaring done. Any of these works: the Claude in Chrome tools, a preview MCP, or a headless browser you already have.

1. Serve the output dir: `python -m http.server 8791 --directory <out-dir>` and open `http://127.0.0.1:8791/index.html`.
2. In the page console, read `getBoundingClientRect().left` for `.head .ey`, `.hero .num`, `.hero .stats .n`, `.dc`, `.clock .cap`, `.era .cell`:

   ```js
   const L = s => Math.round(document.querySelector(s).getBoundingClientRect().left);
   ({ ey:L('.head .ey'), num:L('.hero .num'), stat:L('.hero .stats .n'), dc:L('.dc'),
      clock:L('.clock .cap'), era:L('.era .cell'),
      tmTops:[...document.querySelectorAll('.era .cell .tm')].map(e=>Math.round(e.getBoundingClientRect().top)) })
   ```

   **All tile-content left edges must match.** The header eyebrow sits one tile-padding (~49px) further left, flush with the tile's outer edge — that is correct, not a bug. All era `.tm` tops must be equal (labels must not wrap → keep them one word).
3. Fix any mismatch in `template.html` (shared padding, `white-space:nowrap` on era labels), regenerate, re-verify. Edits do not hot-reload — restart the server or hard-reload. Then stop the server.

## Step 5 — hand off
Tell the user the file path and that they should open it full-screen (browser ≥ 1080px wide) and screenshot for a clean 9:16 share. Never use emojis anywhere on the card — it is part of the card's house style. Icons are inline SVG or typographic only.

## Notes / gotchas
- The card is a **fixed 1080px design** (px units). `.fit` must stay a plain block with `.stage{margin:0 auto}` — never `display:flex` on the wrapper, or the stage shrinks and cramps on narrow viewports.
- The clock bars use a fixed-scale gradient (`background-size:100% 210px; background-position:bottom`) so taller bars reveal more of the clear→orange→black ramp. Keep that technique if you re-theme.
- `--all` dedupes commits across branches; counts each commit once.
