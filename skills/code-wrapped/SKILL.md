---
name: code-wrapped
description: Generate a shareable "Code Wrapped" stats card (Spotify-Wrapped style) from a git repo's history — lines written, commits, files, focused hours, a 24-hour coding clock, a docs-vs-code split, and an era time-machine (how long the same work would have taken in 2000/2010/2020/Nov 2025 vs now). Use when the user asks for a "code wrapped", "coding wrapped", "git wrapped", a coding-stats card, or a year/sprint-in-review graphic for a repository.
---

# Code Wrapped

Produces a self-contained, Instagram-shareable HTML card (FleetMind palette: navy `#0b1c3f`, off-white `#f4f3ee`, orange `#FF5B1F`, black `#0a0a0a`) at 1080px wide. The card sizes to its own content (roughly 9:16) and renders on a seamless navy body — no scrolling, no dead space.

## Inputs
- **repo** — absolute path to a git repo. Default: current working directory.
- **name** — the eyebrow over the title (e.g. `NIKO`). Ask if unknown.
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

5. **Hour-of-day distribution** (24 counts, local time):
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
- **CLOCK_BARS**: the 24 bars as **static HTML**, one div each, height in px. With `BAR_H = 176` and `mx = max(hours)`:
  `<div class="hr" style="height:{max(5, round(v / mx * 176))}px"></div>` joined with no separator.
  Emit real HTML here, not a JS array — the bars must survive a page where scripts do not run.
- **PEAK**: the busiest hour and its count, e.g. `Peak 8pm &middot; 105 commits`. Index of the max in the array; `0` is 12am, `12` is 12pm.

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

Copy `template.html` (next to this file) to `out`, substituting every `{{PLACEHOLDER}}`. Create the output dir if needed. Placeholders: `NAME, LINES, LINES_SUFFIX, FILES, COMMITS, HOURS, CODE_PCT, DOCS_PCT, CLOCK_BARS, PEAK, ERA_2000, ERA_2000_U, ERA_2010, ERA_2010_U, ERA_2020, ERA_2020_U, ERA_NOV, ERA_NOV_U, ERA_NOW, ERA_NOW_U, KICKER_A, KICKER_B`. Assert none are left unfilled before writing.

## Step 4 — verify fit and alignment (do not skip)

The card is a fixed 1080x1920 box. Content that overruns it is silently clipped, so this step is a **numeric assertion, not an eyeball pass**. Serve the output dir (`python -m http.server 8791 --directory <out-dir>`) and run this in the page console:

```js
const st = document.getElementById('stage'), T = st.getBoundingClientRect().top;
const L = s => Math.round(document.querySelector(s).getBoundingClientRect().left);
({
  overflow: st.scrollHeight - 1920,                       // MUST be 0
  bars: document.querySelectorAll('.hr').length,          // MUST be 24
  edges: [L('.hero .num'), L('.hero .stats .n'), L('.dc'),
          L('.clock .cap'), L('.era .cap'), L('.era .cell')],   // MUST all match
  tmTops: [...document.querySelectorAll('.era .cell .tm')]
            .map(e => Math.round(e.getBoundingClientRect().top)) // MUST all match
})
```

- `overflow` must be **0**. Anything positive is clipped content; anything negative is dead space. Fix by adjusting the shared paddings and label sizes, not by growing the stage.
- All tile-content left edges must match. The header eyebrow sits one tile-padding (~45px) further left, flush with the tile's outer edge — that is correct, not a bug.
- All era `.tm` tops must be equal. If one differs, a label wrapped: keep era labels to one word.

Then check it at phone width, in an iframe so you do not have to resize the window:

```js
const f = document.createElement('iframe');
f.style.cssText = 'position:fixed;top:0;left:0;width:390px;height:844px;border:0;z-index:9999';
f.src = '/index.html';
document.body.appendChild(f);
await new Promise(r => f.onload = r);
const d = f.contentDocument;
const out = {
  hScroll: d.documentElement.scrollWidth > d.documentElement.clientWidth,  // MUST be false
  pageH: d.body.scrollHeight,                                              // ~667, MUST be <= 844
  dead: d.body.scrollHeight - Math.round(d.getElementById('fit').getBoundingClientRect().height) // MUST be 0
};
f.remove(); out;
```

Edits do not hot-reload — restart the server or hard-reload before re-checking. Stop the server when done.

Finally, render the shareable PNG straight out of headless Chrome at the exact card size:

```
chrome --headless=new --disable-gpu --hide-scrollbars --force-device-scale-factor=1   --window-size=1080,1920 --screenshot=card.png <out>/index.html
```

Confirm the PNG is 1080x1920 and that the clock bars actually have height in it.

## Step 5 — hand off
Hand over both files: `index.html` (opens on any phone, scales to the screen) and `card.png` (1080x1920, ready to post as-is). The PNG is usually what they actually want — no screenshotting required. Never use emojis anywhere on the card (hard project rule) — icons are inline SVG or typographic only.

## Notes / gotchas
- The card is a **fixed 1080x1920 design** (px units). `.fit` must stay a plain block; the scale script sets its height so there is never dead space under the scaled card. Never put `display:flex` on the wrapper, or the stage shrinks and cramps on narrow viewports.
- The stage uses `justify-content:space-between`, so leftover vertical room spreads evenly between tiles instead of pooling above the kicker. If you add a section, re-run Step 4 — the slack absorbs small additions silently until it does not.
- Every hero stat column is centred. Edge-aligning the outer two floats each label away from its own number, which reads as a broken column once the card is scaled down to a phone.
- The clock bars use a fixed-scale gradient (`background-size:100% 210px; background-position:bottom`) so taller bars reveal more of the clear→orange→black ramp. Keep that technique if you re-theme.
- `--all` dedupes commits across branches; counts each commit once.
