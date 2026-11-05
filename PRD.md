# PRD — Woven Monopoly Simulator

## 1. Overview

A deterministic command-line simulation of a simplified Monopoly variant ("Woven Monopoly"). Dice rolls are pre-determined and supplied as input, so a given board + rolls pair always produces the same result. The program plays a full game and reports the outcome.

## 2. Goals

- Correctly simulate the game as described in the rules, including the documented assumptions below.
- Work on any valid board/rolls file, not just the provided samples — no hard-coded sizes, names, or colours.
- Be clean, tested, and easy to extend, since the rules are expected to change over time.

## 3. Game Rules (summary)

- 4 players, fixed turn order: Peter, Billy, Charlotte, Sweedal.
- Each starts with $16, on GO.
- Passing/landing on GO (after the start) pays $1 per lap (a single big move can cross it more than once — see Assumption #9).
- Landing on an unowned property forces a purchase at its listed price.
- Landing on an owned property requires paying rent to the owner.
- Rent doubles if the owner holds every property in that colour group.
- No chance, jail, or stations.
- Board wraps around past the last space.
- Game ends when a player goes bankrupt; the richest remaining player(s) win.

## 4. Assumptions (gaps the rules leave open)

1. **Can't afford a mandatory purchase** → player pays the bank whatever they have, balance hits $0, declared bankrupt.
2. **Can't afford rent** → player pays the owner whatever they have, balance hits $0, declared bankrupt.
3. **Bankruptcy ends the game immediately** → no further turns/rolls are processed once it happens.
4. **Bankrupt player's properties are irrelevant** → game is already over, so no reassignment logic is needed.
5. **Space behaviour is driven by a `type` field** → only `"property"` spaces trigger buy/rent; any other/unknown type (including `"go"`) is a no-op *landing* space. The GO $1-per-lap bonus is a separate mechanism — it's computed from crossing index 0 arithmetically (Assumption #9), not from a "landing on type == go" effect — so it doesn't conflict with GO otherwise being a no-op space to land on.
6. **Ties are reported as multiple winners** → `winners` can contain more than one name. (Not really a gap — the README states this directly — kept here for completeness since it affects implementation.)
7. **Rolls run out with no bankruptcy** → report the current state of all players, but mark the result as unfinished ("Continued" / "To Be Continued") instead of declaring a winner.
8. **Rent amount** → the board format has no separate rent value (only `price`), and real Monopoly's rent isn't a clean function of price anyway. Simplest consistent rule: **rent equals the property's price** (doubled under the full-colour-set rule). Avoids fractional-dollar issues a ratio like price÷2 would introduce, at the cost of being a less "realistic" curve — acceptable since the board data doesn't support a realistic curve regardless.
9. **A single move can loop the board more than once** → on boards smaller than the maximum roll value, a big enough roll could pass GO multiple times in one move. Pay **$1 per lap** (`(old_position + roll) / board_length`, integer division), not a flat $1 per turn — the literal, consistent reading of "you get $1 when you pass GO."

## 5. Functional Requirements

1. Load a board file (ordered, wrapping list of spaces; properties carry `price`/`colour`).
2. Load a rolls file (ordered list of dice values, any length).
3. Initialize 4 players in fixed order, $16 each, starting at GO.
4. Execute turns: consume next roll, move player, wrap around the board.
5. Pay $1 per lap when passing/landing GO (excluding the initial starting position; Assumption #9 covers multi-lap moves).
6. Force a purchase when landing on an unowned property.
7. Charge rent when landing on an owned property.
8. Double rent when the owner holds a full colour set (rent = property price per Assumption #8).
9. Handle insufficient funds per Assumptions #1/#2 → bankruptcy.
10. Stop the game immediately on bankruptcy (Assumption #3).
11. Determine winner(s) as the player(s) with the most money remaining, supporting ties.
12. Handle roll exhaustion with no bankruptcy per Assumption #7.
13. Treat non-`"property"` space types as no-ops (Assumption #5).
14. CLI: accept `<board_path> <rolls_path> [--json]`.
15. `--json` output: `{ "winners": [...], "players": [{ "name", "money", "space" }, ...] }` (extra fields allowed).
16. Default output: human-readable summary of the same information.
17. Provide a hand-authored `rolls_3.json` (10–30 rolls) with a worked-out expected result.

## 6. Non-Functional Requirements

- **Deterministic** — same inputs always produce the same output.
- **General** — no assumptions baked in from the sample board/rolls.
- **Extensible** — clear separation of concerns so new rules are easy to add later.
- **Readable** — clean, idiomatic Ruby.
- **Testable** — game engine logic unit-testable independent of CLI/file I/O.
- **Ruby 3.1-compatible** (per README — their JRuby production target) even though local dev machine runs Ruby 4.0.6. Install Ruby 3.1.x via a version manager (rbenv/asdf/rvm) alongside 4.0.6 and run the real test suite under 3.1 before calling it done, rather than eyeballing syntax while coding against 4.0.6. Pin via `.ruby-version` and `ruby "~> 3.1"` in the Gemfile. Minimal dependencies (RSpec for tests; RuboCop optional for linting, `TargetRubyVersion: 3.1`).
- **No UI/DB/web framework** — pure CLI.
- **Small, frequent commits** — git history should show incremental progress.
- **Documentation** — README (setup/run/test instructions, design decisions, Assumptions, sample output) + `AI_USAGE.md` (+ `ai-sessions/` if exported).

## 7. Architecture

Four layers, kept separate so logic stays testable and extensible:

1. **CLI entry point** — parses args/flags only.
2. **Loaders** — parse board JSON and rolls JSON into plain Ruby structures; validate shape.
3. **Game engine** — pure simulation logic (movement, buying, rent, bankruptcy, win detection); no knowledge of files or stdout.
4. **Output formatter** — renders engine results as JSON or human-readable text.

No diagram needed — it's a small, single-language CLI project.

## 8. User Flow (input → output)

1. Run `bin/woven_monopoly board.json rolls.json [--json]`.
2. Load + validate board and rolls files.
3. Initialize game state (4 players, $16, GO).
4. Play turns in fixed order, one roll per turn, until bankruptcy or rolls exhausted.
5. Format final state (JSON or readable) and print to stdout.
6. Exit 0 on a normal run (win or continued); non-zero only on genuine input errors (missing file, invalid JSON, malformed structure).

## 9. Entities

No database — these are plain in-memory Ruby objects, kept deliberately minimal. Two entities, no join table:

- **Player** — `name`, `money`, `position` (current space index). One player always has exactly one current position, so it's a plain attribute, not a relationship.
- **Space** — `position` (index, doubles as the board's `position_id`), `name`, `type`, and — only when `type == "property"` — `price`, `colour`, and a mutable `owner` (nil until bought, set to a player reference during play).

Two things deliberately **not** modeled as entities, and why:
- **Player ↔ Space landing history.** Conceptually many-to-many over a full game (a player can land on the same space repeatedly; a space is landed on by multiple players), but nothing in the rules reads history — "has looped" is recomputed arithmetically every turn from `position + roll` vs. board length (Assumption #9), not looked up from a log. No current rule needs a stored record of past landings, so none is kept. (If a future rule needed e.g. "rent increases per repeat visit," this would become a real append-only entity at that point — not before.)
- **Ownership as a separate join entity.** Property ownership is a real one-player-to-many-properties relationship, but since each property has at most one owner, it's cheap enough to carry as the `owner` field directly on `Space` rather than a standalone mapping structure.

## 10. Out of Scope

- Any UI, database, web framework, or persistence.
- Chance cards, jail, stations, mortgaging/selling, auctions.
- Anything not explicitly in the rules or the assumptions above.

## 11. Open Items (not yet decided)

- Exact behaviour on malformed/missing input files beyond "non-zero exit" (e.g. error message format) — to be settled during implementation or treated as a minor detail.

## 12. Deliverables Checklist

- [ ] Ruby source + RSpec tests, verified under Ruby 3.1 (not just local 4.0.6)
- [ ] README: exact run command at the **top**; install/run/test instructions; design decisions; Assumptions section (each with a reason); sample output for rolls_1/rolls_2; "what I'd do next" note if we go over the 2–3hr time-box
- [ ] `AI_USAGE.md` — tracked *during* implementation (tools used, ≥2 concrete cases AI was wrong/unhelpful and how it was caught/fixed, something written/rewritten by hand and why), not reconstructed afterward
- [ ] `ai-sessions/` exports, personal info scrubbed first (if the tool supports export)
- [ ] `rolls_3.json` — expected result worked out by hand *before* running the program, then explained in README
- [ ] Git history with small, incremental commits
- [ ] Final submission includes the `.git` folder (zip or repo link), not just source files
