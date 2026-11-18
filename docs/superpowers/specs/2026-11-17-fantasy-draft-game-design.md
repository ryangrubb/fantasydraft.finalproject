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

- **Leaking the hidden values during the draft**, directly or indirectly. For example, listing available players in order of their 2026 average, or showing this run's nudged board values, would give the answers away. Sorting by real ADP is fine: that's public, and it's what any draft app shows.
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
- **During the draft:** the player sees name, position, NFL team and real ADP. Available players are listed in ADP order, like a real draft app.
- **Computer teams draft like real drafters, using 2026 ADP (average draft position).** At any pick, the pool should look realistic. For example, a consensus top-3 pick like Jahmyr Gibbs is gone by pick 5.
- **Who goes before you varies from run to run**, so the pool at your pick isn't always the same. Stars still go about where they really went, and later rounds vary more.

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
- `adp`: 2026 preseason ADP, 12-team full PPR, used only by the computer teams. Players outside the ADP list get `adp` 200.

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

**ADP source:** a consensus 2026 preseason ADP for 12-team full PPR redraft leagues (e.g. FantasyPros consensus, or Sleeper). Use one source, and note which in a comment in the startup passage.

**Pool** *(Claude's call)*: about 150 players, built from two parts:
1. **The top ~150 by ADP** (QB, RB, WR and TE only). This makes the pool at each pick look like a real draft.
2. **Breakouts:** anyone outside that list whose 2026 average would put him in the top ~150 by value over replacement. Value is measured against the 12th QB, 24th RB, 30th WR and 12th TE. These players get `adp` 200, so the computer teams rarely take them early. They are the reward for knowing who's producing this year.

If both parts together go well over 150, trim the lowest-ADP part-1 players who also have low averages.

Computing this offline with a small script is fine. The game itself only reads `adp`.

**Tuning numbers** live together in the same startup passage: injury odds, IR length, number of rounds, number of teams, and the ADP wobble range.

### Intro

- Keep the existing Opening passage.
- Keep the draft-position roll (1–12).
- **Fix the ordinal:** show "7th pick", not "7 pick" (1st, 2nd, 3rd, 4th–12th).
- Show the snapshot line.
- Show the current high score, or "No high score yet".

### Draft

**Order:** snake. Team 1 picks first in odd rounds and last in even rounds. The player is team `$draftPosition`.

**Draft board for this run** *(Claude's call)*:
- At the start of each run, give every player a **board value = `adp` × a random factor between 0.85 and 1.15**, rolled fresh each run.
- The computer teams always take the available player with the **lowest board value**.
- Because the wobble is a percentage, it stays small for stars (ADP 2 lands between 1.7 and 2.3) and grows later on (ADP 60 lands between 51 and 69).
- So the first picks stay realistic, a top-3 player is gone by pick 5, and who's left at your mid- and late-round picks changes from run to run.
- The ±15% range is a tuning number in the startup passage.

**Roster rules per computer team:**
- **No 2nd QB or 2nd TE before round 8**, and at most 2 of each. When a team takes its first QB or TE is left to ADP, so elite ones like Josh Allen, Brock Bowers and Trey McBride go early and the rest don't.
- If a candidate would break a rule, skip him and take the next-lowest board value.

**Running computer picks:**
- Computer picks run without the player clicking through them, until it's the player's turn.
- Show a short "Picks since your last turn" list (pick number and player name only), so the player sees who went.

**Player's pick:**
- Show the available players in **one list sorted by real `adp`**, lowest first, the way a draft app does *(author's decision)*. Never sort by `avg` or by this run's board value (see Anti-goals).
- Breakout players (`adp` 200) go at the bottom in alphabetical order, with their ADP shown as "—".
- Each entry shows ADP, name, position, team, and a link to draft him *(showing the ADP number is Claude's call)*.
- **Position filter** *(Claude's call)*: links at the top (All / QB / RB / WR / TE) narrow the list, so 150 names stay usable.
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
