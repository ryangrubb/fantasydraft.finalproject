# Fantasy Draft

An interactive fantasy football draft game built in [Twine](https://twinery.org/) with the Harlowe 3.3.9 story format.

## Layout

| Path | What it is |
|------|------------|
| `src/fantasy-draft.twee` | The story as plain text (Twee 3). Diff-friendly source of truth. |
| `published/fantasy-draft.html` | The playable game, as published from Twine. Open it in a browser. |
| `.claude/skills/` | [Superpowers](https://github.com/obra/superpowers) skills for Claude Code (MIT, see `.claude/skills/LICENSE`). |

## Working with the story

- **In Twine:** Library → Import → choose `published/fantasy-draft.html` (or the `.twee` file). After editing, use Publish to File and save over `published/fantasy-draft.html`.
- **From the command line:** [Tweego](https://www.motoslave.net/tweego/) compiles the Twee source: `tweego -o published/fantasy-draft.html src/fantasy-draft.twee`.

Keep the `.twee` file and the published HTML in sync when committing, so changes show up as readable diffs.
