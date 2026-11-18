# Fantasy Draft Game — Design

**Date:** 2026-11-17
**Status:** Awaiting approval
**Repo:** `ryangrubb/fantasydraft.finalproject`
**Source:** `src/fantasy-draft.twee` (Harlowe 3.3.9, Twine 2.12)

## Intent

A replayable fantasy football draft game built in Twine. The player drafts one team against 11 computer teams, and the moment the draft ends the game scores the team and shows it next to the player's best score so far. The fun is "one more try": beat your high score.

Two things drive the score:

- **Skill:** football knowledge. Player values are hidden during the draft. You pick on what you know about who is actually producing in the 2026 season.
- **Luck:** where you pick, whether the computer teams leave a star on the board, and who gets hurt during the simulated season.

The current story is the start of this game: an opening passage, a random draft position from 1 to 12, and an empty `DraftRound1` passage.

## Goals

- One full draft and its result take a few minutes, so replaying feels natural.
- Draft decisions matter: positions, the starting lineup, bye weeks and bench depth all affect the score.
- Scores vary from run to run but skill visibly wins over time.
- The high score survives closing the browser.
- The player list and tuning numbers are easy to find and edit in the story file.

## Non-goals

- Head-to-head matchups, standings or playoffs.
- Kickers and team defenses.
- Trades, waivers, free agents, or setting your own lineup each week.
- Live or automatically updated stats. The data is a frozen snapshot.
- Different injury risk for different players or positions.

## Anti-goals (would make it a failure even if it works)

- **Leaking the hidden values during the draft**, directly or indirectly. For example, listing available players in value order would give away the rankings.
- **A slog:** too many clicks per run, or long waits while computer teams pick.
- **Luck swamping skill**, so a well-built team can't reliably beat a careless one.

## Constraints

- **Story format:** Harlowe 3.3.9, which is what the existing story uses. No hacks that reach into Harlowe from JavaScript.
- **Files:** keep `src/fantasy-draft.twee` (readable source) and `published/fantasy-draft.html` (playable game) in sync.
- **Data:** real 2026 NFL players, using **full PPR scoring for a 12-team league**.

## Approaches considered

1. **Stay in Harlowe (chosen).** All data and logic go in Harlowe passages, and a save-game slot holds the high score. It keeps the existing passages, styling and the author's Harlowe knowledge, and the project stays editable in Twine. The cost is that Harlowe loops are wordy, but the maths involved is small.
2. **Switch to SugarCube.** It has JavaScript-like scripting and built-in persistent storage. It was not chosen because the existing work would need rewriting and the project would leave the format the author knows.
3. **Harlowe plus a plain-JavaScript engine.** Rejected because Harlowe blocks JavaScript from reaching its state, so it would depend on fragile hacks.

## Decided by the author

- **Game loop:** draft, then an immediate score, then try again for a high score.
- **Starting lineup:** 1 QB, 2 RB, 2 WR, 1 TE, 1 FLEX (RB, WR or TE).
- **Bench:** it matters, because it covers bye weeks and injuries.
- **Hidden value:** each player's 2026 **average PPR points per game** so far.
- **Weekly scoring:** "option A". Each week a player scores his average, or 0 if he's on bye or injured.
- **Season score:** the average of the weekly team totals across the season.
- **Random injuries:** three severities (1 week missed, IR, season-ending), each starting in a random week.
- **Injury odds:** the same for every player.
- **Real-life injuries and status:** ignored. Only the game's random injuries count.
- **During the draft:** the player sees only name, position and NFL team.
- **Computer teams:** they usually take the best available player, but a randomizer sometimes lets good players slide.

## Calls made in the conversation (proposed by Claude, not objected to)

- **Draft size:** 10 rounds (7 starters + 3 bench), 12 teams, so 120 picks.
- **Draft order:** snake.
- **Lineup:** the game sets the best possible lineup for you each week.
- **High score:** stored using Harlowe's save-game slot name.
- **Player pool:** about 150 players.
- **Season length:** 17 weeks are simulated.
- **IR length:** 4 weeks.
- **Starting injury odds (per player, per season, mutually exclusive):**

  | Injury | Chance |
  |--------|--------|
  | 1 week missed | 15% |
  | IR (4 weeks) | 8% |
  | Season-ending | 4% |

- **Data date:** a frozen snapshot, shown on screen.
- **Results screen:** shows weekly totals, injuries, your roster with values revealed, the season average and the high score.

## How it works

### Data

One passage tagged `startup` sets up the player list: an array of datamaps.

Each player has:
- `name`, `pos` (QB/RB/WR/TE), `team`
- `avg`: 2026 PPR points per game, one decimal
- `bye`: 2026 bye week
- `rank`: overall draft rank, used only by the computer teams

**Snapshot:** through the **latest fully completed week** when the data is collected (Week 10 at the time of writing). The week number is stored as one variable in the startup passage, and the intro screen reads from it ("Player values: 2026 season, Weeks 1–10, full PPR"). *(Claude's call.)*
- If complete data for that week isn't published yet, use the previous complete week.
- Never use estimated or invented averages in the finished game.
- A small made-up test list is fine while building the logic, as long as it's replaced before the game is finished.

**Building the list:**
- Use any reputable source of season PPR totals and games played.
- `avg = total PPR points ÷ games played`. Only count players with at least 3 games played.
- **Real-life status is ignored** *(author's decision)*. Anyone with 3+ games played can be in the pool, even if he is currently injured, suspended or released. His frozen average is used, and only the game's random injuries apply.
- A traded player shows his current team *(Claude's call)*.
- Bye weeks come from the official 2026 NFL schedule.

**Pool:** roughly 20 QB, 50 RB, 60 WR and 20 TE, chosen by `rank`. The aim is a pool comfortably larger than 120 picks, with realistic depth at each position.

**`rank` measures value over a replacement player at the same position.** It is not raw average, because QBs score the most raw points and the computer teams would otherwise take QBs first.
- `value = avg − baseline`. The baseline is the average of the 12th-best QB, 24th RB, 30th WR and 12th TE.
- Sort by `value`, highest first. That order is `rank`.
- Computing this offline with a small script is fine. The game itself only reads `rank`.

**Tuning numbers** live together in the same startup passage: injury odds, IR length, number of rounds, number of teams, and the computer randomness weights.

### Intro

- Keep the existing Opening passage.
- Keep the draft-position roll (1–12).
- **Fix the ordinal:** show "7th pick", not "7 pick" (1st, 2nd, 3rd, 4th–12th).
- Show the snapshot line.
- Show the current high score, or "No high score yet".

### Draft

**Order:** snake. Team 1 picks first in odd rounds and last in even rounds. The player is team `$draftPosition`.

**Computer pick:**
- Look at the 3 best available players by `rank`.
- Take the best 60% of the time, the 2nd 25%, and the 3rd 15%.
- **Roster limits per computer team:** at most 2 QB and 2 TE. If a candidate would break a limit, skip him and look further down the list. This stops the computer teams from wasting picks in ways that look silly.

**Running computer picks:**
- Computer picks run without the player clicking through them, until it's the player's turn.
- Show a short "Picks since your last turn" list (pick number and player name only), so the player sees who went.

**Player's pick:**
- Show the available players **grouped by position**, sorted **alphabetically** within each group. Never sort by `avg` or `rank` (see Anti-goals).
- Each entry shows name, team, and a link to draft him.
- Show the player's roster so far.
- Show the round and pick number.

**Draft end:** after the player's 10th pick, go straight to the season. Computer picks after the player's last pick don't need to happen, because the computer teams are never scored.

### Season simulation (player's team only)

1. **Roll injuries for each of the 10 rostered players**, using a single roll per player:
   - **1–4 (4%):** season-ending, from week W through Week 17
   - **5–12 (8%):** IR, out weeks W to W+3, cut off at Week 17
   - **13–27 (15%):** 1 week missed, out week W only
   - otherwise healthy

   W is random from 1 to 17.

2. **For each week from 1 to 17,** a player is available if it's not his bye week and he isn't injured. Fill the lineup greedily from available players:
   - the best QB
   - the best 2 RBs
   - the best 2 WRs
   - the best TE
   - FLEX: the best remaining RB, WR or TE

   With these slots, filling greedily always gives the best possible lineup. An empty slot scores 0.
   **Weekly total** = the sum of the starters' `avg`.

3. **Season score** = the average of the 17 weekly totals, rounded to 1 decimal.

### Results

- **Headline:** the season score ("Your team averaged 131.6 points a week").
- **High score:** the previous high score, and "NEW HIGH SCORE!" if this run beat it.
- **Roster:** the 10 players with their revealed averages and bye weeks.
- **Injuries:** what happened, e.g. "Week 9: [player] — season-ending".
- **Weekly totals:** a compact list or table, Weeks 1–17.
- **Draft again:** a link that starts a fresh run.

### High score storage

- `(save-game: "best", <score as text>)` stores the score as the slot's file name.
- `(savedgames:)'s best`, converted with `(num:)`, reads it back, if the slot exists.
- Only save when the new score is higher.

## Left to the builder

- Passage names and structure, and how the draft loop is split across passages.
- Exact wording, layout and styling, as long as it fits the existing dark theme.
- Whether "Draft again" uses `(restart:)` or resets variables in place. The high score must survive either way.
- The data source, and how the list is assembled (by hand or with a script).
- The build tooling. Twine import/publish or Tweego are both fine, as long as both files stay in sync.
