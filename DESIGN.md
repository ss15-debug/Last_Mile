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
care calls, rating) and picks one of three actions:

| Action | Cost | Effect |
|---|---|---|
| **Reject** | -$10 restocking fee | order goes away, no risk |
| **Ship** | free | outcome comes from the data |
| **Expedite** | 1 token (5 per shift) | guaranteed on time |

Each action is correct in a different situation, which is what makes it a
decision rather than a button:

- order looks doomed, tokens left -> **Expedite**
- order looks doomed, out of tokens -> **Reject** (lose $10 instead of $50)
- order looks ordinary -> **Ship** (save the token for something worse)

Tokens are the scarce resource. There are 5 per shift and roughly 5.4 doomed
orders in an average 20-order shift, so you cannot expedite your way out —
some doomed orders have to be rejected instead.

## 4. Scoring

| Event | Money |
|---|---|
| Order arrives on time | +$40 |
| Order arrives late | -$50 |
| Reject an order | -$10 |
| Expedite | costs a token, not cash |

**Target to win a shift: $100.**

These numbers were tuned by simulating 3,000 shifts per configuration against
the real live-order pool (`scratchpad/balance2.py` methodology, reproduced in
`analysis.py` in Phase 2). Average result per 20-order shift:

| Strategy | Result |
|---|---|
| Ship everything (knows nothing) | **-$243** |
| Expedite at random | -$3 |
| Found one rule | **+$159** |
| Found both rules | **+$197** |

A player who knows nothing loses badly. A player who guesses breaks even. Each
rule discovered is worth real money. That progression is the game working.

Note that the second rule is worth less than the first (+$38 vs +$162),
because the discount rule covers more orders. That is fine — diminishing
returns on investigation is realistic.

## 5. The Notebook (the learning mechanic)

The player cannot win by guessing, so the game gives them a Notebook. Per the
wireframe (`docs/Notebook_Picture.png`) it is a two-page spread with tabs:

- **LOGS** — every order from past shifts, with the action taken and outcome
- **NOTES** — a free-text scratchpad for the player's own theories
- **QUERY** — pick a column, see late rate grouped by it, as a table or bar chart

QUERY is where rules get discovered. Grouping by `Discount_offered` shows a
flat line at ~46% that jumps to 100% at 11 and never comes back down.

The Notebook reads only the **archive**, never live orders — see section 6.

## 6. The split (and why it must be shuffled)

- **archive** (8,000 orders) — queryable in the Notebook
- **live pool** (2,999 orders) — dealt during shifts, never queryable

This is a train/test split: study the past, predict the future, no cheating by
looking up the order in your hand.

**The file must be shuffled before splitting.** `Train.csv` is ordered — the
first ~3,000 rows are 100% late and ~93% doomed, while everything after row
4,000 is a near-pure coin flip. A naive sequential split put only 2 doomed
orders into a live pool of 2,999, making the game unwinnable by skill: expert
play scored identically to knowing nothing.

Shuffle with a fixed seed so the split is reproducible:

```python
rng = np.random.default_rng(42)
shuffled = df.iloc[rng.permutation(len(df))].reset_index(drop=True)
archive, live = shuffled.iloc[:8000], shuffled.iloc[8000:]
```

After shuffling, the live pool is 27.2% doomed — matching the file overall.
This is a hard requirement, and `tests/` must assert it in Phase 4.

## 7. Screens

Wireframes: `docs/Shift_Screen.png`, `docs/Notebook_Picture.png`.
Visual style is bureaucratic terminal — monospace text, box-drawing borders,
no sprite art. This is an aesthetic choice and also a scope choice: the entire
UI is rectangles and text, which removes the art pipeline from Phase 3.

1. **Title** — name, Play, Notebook, Quit
2. **Shift** — see below
3. **Reveal** — outcome stamped on the card, ledger updates
4. **Notebook** — tabbed: LOGS / NOTES / QUERY
5. **Shift summary** — final ledger, win or lose, next shift

### Shift screen layout

Top bar: ledger total on the left, remaining tokens on the right.
Centre: the order card. Bottom: ledger quick-panel, then three buttons.

The order card shows the raw data fields, unhighlighted — the fields ARE the
clues, and the game never points at them:

    +--------------------------------+
    | ORDER #1042                    |
    | ------------------------------ |
    | Warehouse ....... F            |
    | Mode ............ Ship         |
    | Weight .......... 3,088 g      |
    | Cost ............ $216         |
    | Discount ........ 59%          |
    | Importance ...... low          |
    | Prior orders .... 2            |
    | Care calls ...... 4            |
    +--------------------------------+

        [ REJECT ]  [ SHIP ]  [ EXPEDITE ]

That example is a real row from the dataset, and it is doomed twice over —
discount 59 and weight 3,088 both trip a rule. A new player cannot see that.

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
