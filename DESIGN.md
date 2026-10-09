This containes game rules, scoring, screens, and architecture

# Last Mile — Design Document

> Status: Phase 1 (design locked, not yet built)
> Author: Shreyas Sahoo

## 1. Premise

You are a dispatcher at a distribution center. Every shift, 20 orders cross your
desk. Some of them are doomed to arrive late — and the warehouse has never told
you why.

You have 5 expedite tokens per shift. Spend them on the right orders and you finish in profit. Spend them wrong and you eat the penalties.

Nobody tells you the rules.

## 2. Why this design

The dataset (`Train.csv`, 10,999 shipping records) turned out to be **synthetic** (AKA AI generated), with two deterministic rules baked into the outcome label:

- `Discount_offered > 10`  -> late 100.00% of the time (n=2,647)
- `Weight_in_gms` in 2000-4000g -> late 99.83% of the time (n=1,792)

Together those cover 26.5% of all orders. The other 73.5% are a 45.1% coin flip, and every remaining column (warehouse, shipping mode, gender, rating) is noise.

An earlier design had the player choosing shipping modes against historical
odds. That design is impossible here: shipping mode moves the late rate by only, 1.4 points, so the choice would be meaningless.

This design turns the flaw into the whole point. The hidden rules ARE the game.

## 3. Core loop

One shift = 20 orders, dealt one at a time.

For each order the player sees every input field (warehouse block, shipping
mode, weight, cost, discount, product importance, prior purchases, customer
care calls, rating) and chooses:

| Choice | Cost | Effect |
|---|---|---|
| **Ship standard** | free | outcome comes from the data |
| **Expedite** | 1 token (5 per shift) | guaranteed on time |

Then the outcome is revealed and the ledger updates.

## 4. Scoring

| Event | Money |
|---|---|
| Order arrives on time | +$20 |
| Order arrives late | -$50 |
| Expedite | costs a token, not cash |

Tokens are the scarce resource. That scarcity is what makes the decision real —
if expediting were free or unlimited, the correct play would be to expedite
everything and there would be no game.

Expected outcomes for a 20-order shift:

- **Random token use:** ~5 tokens x 45% hit rate -> saves roughly $157
- **Perfect play:** all 5 tokens on rule-matching orders -> saves $350
- **Skill gap: about $193 per shift**

Target to win a shift: **$500**.

## 5. The Notebook (the learning mechanic)

The player cannot win by guessing. They need evidence, so the game gives them a
Notebook — a panel where they can ask questions of *historical* orders:

> "Show me the late rate grouped by `Discount_offered`"

The Notebook returns a small table or bar chart. The player forms a hypothesis,
tests it, and eventually spots the cliff at discount 10.

This is deliberately a train/test split, a real machine-learning concept:

- **Rows 0-7999** -> historical archive, queryable in the Notebook
- **Rows 8000-10998** -> live orders, dealt during shifts, never queryable

The player studies the past to predict the future, and cannot cheat by looking
up the answer to an order they are currently holding.

## 6. Win / lose

- **Win a shift:** finish at $500 or more
- **Lose a shift:** finish below $0
- **Campaign:** 5 consecutive shifts; final score is the total

A first-time player should lose. A player who has found one rule should break
even. A player who has found both should win comfortably. If that progression
does not happen in playtesting, the numbers in section 4 need retuning.

## 7. Screens

1. **Title** — name, Play, Notebook, Quit
2. **Shift** — order card, ledger, token counter, two action buttons
3. **Reveal** — outcome animation, running total
4. **Notebook** — column picker, result table / bar chart
5. **Shift summary** — final ledger, win or lose, next shift

## 8. Architecture

The rule that makes everything else possible: **logic never imports the UI, and
the UI never contains rules.**

    game/
      logic.py   # rules, scoring, dealing orders. No pixels. Fully testable.
      data.py    # loads Train.csv, owns the train/test split
      ui.py      # draws, reads clicks. Knows no rules.
    analysis.py  # Phase 2 exploration, produces docs/ charts
    tests/       # pytest suite against logic.py

`logic.py` must run headless, with no window open. That is what makes Phase 4
testing possible — you cannot write automated tests against something that only
exists as pixels on a screen.

The connection between the two halves is deliberately thin:

    result = logic.play_turn(order, choice="expedite")
    ui.draw_result(result)

## 9. Out of scope for v1.0. I might create the following in a v2.0

Sound, save files, difficulty settings, animated truck routes, leaderboards.
All reasonable later. None of them are the game.
