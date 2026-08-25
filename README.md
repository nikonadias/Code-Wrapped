# Code Wrapped

A Claude Code skill that turns a git repo's history into a shareable, Spotify-Wrapped style stats card.

Point it at a repo. It reads the log, does the math, and writes one self-contained HTML file at 1080px wide, sized for a 9:16 screenshot.

![Code Wrapped card](example/preview.png)

## What it counts

| Stat | How it is measured |
|---|---|
| Lines written | Added lines across `--all --no-merges`, with lockfiles, `node_modules/`, `dist/`, `vendor/`, `.min.` and `.db` filtered out |
| Files | Unique paths ever touched |
| Commits | `git rev-list --all --no-merges --count` |
| Focused hours | Session-gap method. A gap over 90 minutes starts a new session, in-session gaps are summed. Conservative floor, not a guess. |
| Code clock | Commits per hour of day, local time, 24 bars |
| Docs vs code | Conventional-commit prefixes. `docs:` on one side, everything else on the other. |
| Era time machine | The same line count at 2000 / 2010 / 2020 / Nov 2025 tooling rates against 20 person-hours a day, next to what it actually took |

The era row is the point of the card. It puts "this took me 16 weeks" next to "this was 27 years of work in 2000."

## Install

Skills live in `~/.claude/skills/`. On Windows that is `%USERPROFILE%\.claude\skills\`.

```bash
git clone https://github.com/nikonadias/code-wrapped.git
cp -r code-wrapped/skills/code-wrapped ~/.claude/skills/
```

PowerShell:

```powershell
git clone https://github.com/nikonadias/code-wrapped.git
Copy-Item -Recurse code-wrapped\skills\code-wrapped "$env:USERPROFILE\.claude\skills\"
```

Restart Claude Code. Confirm it loaded with `/code-wrapped`.

For a project-local install instead, drop it in `.claude/skills/code-wrapped/` inside the repo.

## Use

```
/code-wrapped
```

Or just ask: "make me a code wrapped for this repo."

Optional inputs:

- **repo** - path to the git repo. Defaults to the current directory.
- **name** - the eyebrow over the title, like `NIKO`.
- **range** - a date window, for example `--since=2026-01-01 --until=2026-12-31`. Defaults to full history.
- **out** - output path. Defaults to `<repo>/scratch/code-wrapped/index.html`.

Then open the file in a browser at least 1080px wide and screenshot it.

## Re-theme it

Everything visual is in the `:root` block at the top of `skills/code-wrapped/template.html`.

```css
--navy0:#102450; --navy1:#0b1c3f; --navy2:#081530;   /* card gradient */
--paper:#f4f3ee;                                      /* text */
--orange:#FF5B1F; --orange2:#FF7841;                  /* the one accent */
```

Swap those five values and the whole card follows. The clock bars use a fixed-scale gradient so taller bars reveal more of the ramp. Keep `background-size:100% 210px; background-position:bottom` if you restyle them.

The card is a fixed 1080px design in px units. Leave `.fit` as a plain block with `.stage{margin:0 auto}`. Setting `display:flex` on the wrapper shrinks the stage and cramps everything on narrow viewports.

## House rules the skill enforces

- No emojis anywhere on the card. Icons are inline SVG or typographic only.
- One accent color. Orange carries the eye, nothing else competes.
- Layout is verified numerically before it is called done, not eyeballed. Step 4 of the skill reads bounding boxes and asserts every tile's content shares a left edge.

## Example

`example/index.html` is a real card, generated from a private repo over 2026-05-07 to 2026-08-25: 781K lines, 3,733 files, 1,466 commits, 188 focused hours.

## License

MIT. See [LICENSE](LICENSE).
