---
name: code-wrapped
description: Generate a shareable "Vibe Code Wrapped" stats card (Spotify-Wrapped style) from a git repo's history — lines written, commits, files, focused hours, a 24-hour coding clock with a labelled y-axis, a docs-vs-code split, and an era time-machine (how long the same work would have taken in 2000/2010/2020/Nov 2025 vs what it actually took with an AI coding tool). Use when the user asks for a "vibe code wrapped", "code wrapped", "coding wrapped", "git wrapped", a coding-stats card, or a year/sprint-in-review graphic for a repository.
---

# Vibe Code Wrapped

Produces a self-contained, Instagram-shareable HTML card at **exactly 1080x1920 — true 9:16**, so one phone screenshot captures the whole thing. Default palette: navy `#0b1c3f`, off-white `#f4f3ee`, orange `#FF5B1F`, black `#0a0a0a`. Re-theme by editing the `:root` block in `template.html`.

Two things make it phone-safe and both are load-bearing:

- **The card scales to the viewport.** A tiny script sets `--s` to `viewportWidth / 1080` and the stage is `transform: scale(var(--s))`, so on a 390px phone the whole card renders at 390x667 with no horizontal scroll and no dead space under it.
- **The clock bars are static HTML with inline heights, not JS-generated.** If the script never runs — file opened over `file://` with scripts blocked, an in-app browser, a saved copy — the card still renders complete with all its data. Only the scaling degrades (you get 1:1 and pinch-zoom). Never move the bars back into JS.

---

## What this skill reads, and what it never does

Read this before running it on someone else's repo, and tell the user what it covers if they ask.

**It reads:** `git log`, `git rev-list`. Commit metadata only — timestamps, subject-line prefixes, changed paths, and added-line counts.

**It never:**
- reads file *contents*, only the numbers in `--numstat`
- writes anything outside the `out` directory
- makes a network request, installs a dependency, or calls an API
- touches the working tree, the index, refs, or any remote — every command is read-only
- reads `.env`, credentials, or anything outside the git history
- uploads, publishes, or posts the card anywhere

**The output is a local file.** The skill hands over a path. Publishing is always the user's own action.

**Say this out loud when the repo is private or a client's**, before rendering — the card is derived data, and derived data still discloses:
- total commits, total lines, file count — i.e. the size and pace of the project
- an hour-by-hour activity histogram — i.e. when that person or team works, including nights and weekends
- the first and last commit dates

None of that is secret code, but on a client engagement it is often commercially sensitive, and the hour histogram is personal information about whoever authored the commits. If more than one person contributed, the card aggregates all of them — the "focused hours" and the clock are the *team's*, not the requester's, and it should not be posted as a personal stat without saying so. Ask before rendering a card for a repo the user does not own.

---

## Step 0 — interview the user first

Ask before running anything. Use the AskUserQuestion tool so these come as pickable options, not a wall of prose. Batch them into one call.

1. **Which AI coding tool should the card credit?** This fills the highlighted final cell of the era grid. Offer: `Claude Code`, `Cursor`, `Copilot`, `Codex`. Keep the label short — it is `white-space:nowrap` in a ~140px cell, so `Copilot` fits and `GitHub Copilot Enterprise` does not. Step 4 catches an overflow.
2. **What name goes above the title?** The eyebrow, e.g. `NIKO`. A handle or a team name works. Offer to leave it blank.
3. **Full history, or a date window?** Default is full history. If they pick a window, append `--since=<date> --until=<date>` to **every** git command in Step 1 — missing one silently mixes scopes.
4. **Which repo?** Default is the current working directory. Confirm it is a git repo and confirm they own it or have the right to publish stats about it.

Do not ask about the output path unless they raise it. Do not ask them to confirm the design.

If the user has already answered any of these in their request, do not re-ask it.

---

## Inputs
- **repo** — absolute path to a git repo. Default: current working directory.
- **name** — the eyebrow over the title (e.g. `NIKO`).
- **tool** — the AI coding tool credited in the final era cell (e.g. `Claude Code`).
- **range** — optional date window. Default: full history.
- **out** — output path. Default: `<repo>/scratch/code-wrapped/index.html`.

---

## Step 1 — gather raw stats (run these in `<repo>`; they're the verified commands)

Use the Bash tool. All use `--all --no-merges` for the full picture. All are read-only.

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

If the repo has no conventional-commit prefixes at all, the split is meaningless — say so and either drop the tile or report it as 100% code rather than inventing a ratio.

---

## Step 2 — derive the display values

- **LINES / LINES_SUFFIX**: round lines to a clean unit. `302341 → 302 / K`; `1.2M → 1.2 / M`. Keep it short.
- **FILES / COMMITS / HOURS**: thousands-separated integers (e.g. `3,010`).
- **RANGE**: the window the card covers, uppercase, e.g. `MAY 7 – AUG 25, 2026`. Use the real first and last commit dates from Step 1, not the dates the user asked for — if the repo's history starts later than `--since`, the card must say what it actually measured. Keep it one line; it is `nowrap`.
- **TOOL**: the tool from Step 0, e.g. `Claude Code`.
- **CLOCK_BARS**: the 24 bars as **static HTML**, one div each, height in px. With `BAR_H = 176` and `mx = max(hours)`:
  `<div class="hr" style="height:{max(5, round(v / mx * 176))}px"></div>` joined with no separator.
  Emit real HTML here, not a JS array — the bars must survive a page where scripts do not run.
- **PEAK**: the busiest hour and its count, e.g. `Peak 8pm &middot; 105 commits`. Index of the max in the array; `0` is 12am, `12` is 12pm.
- **Y_MAX / Y_MID**: the clock's y-axis scale — `max(hours)` and `round(max/2)`. They label gridlines at the top, middle and baseline of the 176px plot, so the reader can price any bar, not just the peak. Because the tallest bar is exactly `BAR_H`, `Y_MAX` sits flush on the peak.

### Era time-machine (the bottom grid)
Answer: *"at 20 hours of code a day (a 2-person team), how long would `LINES` take with each era's tools?"* Use net delivered lines-per-person-hour (LPH) that climbs with tooling, against 20 person-hours/day, every day:

| Era | tooling label | LPH | formula |
|-----|---------------|-----|---------|
| 2000 | by hand | 4 | days = LINES / (4 × 20) |
| 2010 | forums | 10 | days = LINES / (10 × 20) |
| 2020 | frameworks | 21 | days = LINES / (21 × 20) |
| Nov 2025 | AI agents | 83 | days = LINES / (83 × 20) |
| Actual | `{{TOOL}}` | — | the **actual** elapsed time of the range (the span from Step 1) |

These LPH figures are a rough industry-shaped model, not a measurement. The card is a brag, not a benchmark — do not defend the numbers as precise, and do not let the user present them as a study.

Convert days → the cleanest human unit and **round to a clean number** (years if ≥ ~400 days, months if ≥ ~50 days, else weeks). For ~300K lines the canonical set is **10 years · 4 years · 2 years · 6 months · 6 weeks**. Fill `ERA_2000`/`ERA_2000_U` … `ERA_NOW`/`ERA_NOW_U` (value + unit like `years`/`months`/`weeks`).

- **KICKER_A / KICKER_B**: orange lead + muted tail. Default: `More shipped in <actual>` / `than <2000 figure> of hand-coding would have.`

---

## Step 3 — render

Copy `template.html` (next to this file) to `out`, substituting every `{{PLACEHOLDER}}`. Create the output dir if needed. Placeholders:

`NAME, RANGE, TOOL, LINES, LINES_SUFFIX, FILES, COMMITS, HOURS, CODE_PCT, DOCS_PCT, CLOCK_BARS, PEAK, Y_MAX, Y_MID, ERA_2000, ERA_2000_U, ERA_2010, ERA_2010_U, ERA_2020, ERA_2020_U, ERA_NOV, ERA_NOV_U, ERA_NOW, ERA_NOW_U, KICKER_A, KICKER_B`

Assert none are left unfilled before writing.

---

## Step 4 — verify fit and alignment (do not skip)

The card is a fixed 1080x1920 box. Content that overruns it is silently clipped, so this step is a **numeric assertion, not an eyeball pass**. Serve the output dir (`python -m http.server 8791 --directory <out-dir>`) and run this in the page console:

```js
const st = document.getElementById('stage'), T = st.getBoundingClientRect().top;
const L = s => Math.round(document.querySelector(s).getBoundingClientRect().left);
const plot = document.querySelector('.clock .plot').getBoundingClientRect();
({
  overflow: st.scrollHeight - 1920,                       // MUST be 0
  bars: document.querySelectorAll('.hr').length,          // MUST be 24
  gridlines: [...document.querySelectorAll('.clock .gl i')]
    .map(e => Math.round(e.getBoundingClientRect().top - plot.top)),  // MUST be [0, 88, 176]
  headFits: document.querySelector('.head .dt').getBoundingClientRect().right
            <= document.querySelector('.head').getBoundingClientRect().right,  // MUST be true
  clipped: [...document.querySelectorAll('.era .cell .tl, .head .dt')]
    .filter(e => e.scrollWidth > e.getBoundingClientRect().width + 1)
    .map(e => e.textContent),                             // MUST be empty
  edges: [L('.hero .num'), L('.hero .stats .n'), L('.dc'),
          L('.clock .cap'), L('.era .cap'), L('.era .cell')],   // MUST all match
  tmTops: [...document.querySelectorAll('.era .cell .tm')]
            .map(e => Math.round(e.getBoundingClientRect().top)) // MUST all match
})
```

- `overflow` must be **0**. Anything positive is clipped content; anything negative is dead space. Fix by adjusting the shared paddings and label sizes, not by growing the stage.
- All tile-content left edges must match. The header eyebrow sits one tile-padding (~45px) further left, flush with the tile's outer edge — that is correct, not a bug.
- `clipped` must be empty. Both the date and the tool label are `nowrap`, so an over-long value is silently cut rather than wrapped. A long tool name is the most likely trigger — shorten it.
- All era `.tm` tops must be equal. If one differs, a label wrapped: keep era labels to one word.
- The clock gridlines must land on 0 / 88 / 176 and the y labels must centre on them. The axis lives in a `position:relative` 176px plot with the gridlines absolutely positioned behind `.bars`, so it costs no height — if `overflow` moved when you touched the clock, something in there went back into flow.

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
chrome --headless=new --disable-gpu --hide-scrollbars --force-device-scale-factor=1 \
  --window-size=1080,1920 --screenshot=card.png <out>/index.html
```

Confirm the PNG is 1080x1920 and that the clock bars actually have height in it.

---

## Step 5 — hand off

Hand over both files: `index.html` (opens on any phone, scales to the screen) and `card.png` (1080x1920, ready to post as-is). The PNG is usually what they actually want — no screenshotting required.

Never use emojis anywhere on the card — it is part of the card's house style. Icons are inline SVG or typographic only.

---

## Notes / gotchas

- The card is a **fixed 1080x1920 design** (px units). `.fit` must stay a plain block; the scale script sets its height so there is never dead space under the scaled card. Never put `display:flex` on the wrapper, or the stage shrinks and cramps on narrow viewports.
- The stage uses `justify-content:space-between`, so leftover vertical room spreads evenly between tiles instead of pooling above the kicker. If you add a section, re-run Step 4 — the slack absorbs small additions silently until it does not.
- Every hero stat column is centred. Edge-aligning the outer two floats each label away from its own number, which reads as a broken column once the card is scaled down to a phone.
- The clock bars use a fixed-scale gradient (`background-size:100% 176px; background-position:bottom`) so taller bars reveal more of the clear→orange→black ramp. Keep that technique if you re-theme, and keep the number in sync with the plot height.
- `--all` dedupes commits across branches; counts each commit once.
- `--all` also counts commits by **every** author in the history, not just the requester. On a shared repo, say whose numbers these are.
