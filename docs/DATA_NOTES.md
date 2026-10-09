# Data Notes — Last Mile

from my Data 101 class at college, I learned to use the R language. I think it will be interesting to use what I learned to anaylze the data structure

Dataset: [E-Commerce Shipping Data](https://www.kaggle.com/datasets/prachi13/customer-analytics)
File: `Train.csv` — 10,999 rows x 12 columns, zero missing values.

## Gotchas found

**1. The outcome column is inverted.**
`Reached.on.Time_Y.N` is `1` when the order was **late** and `0` when it was
on time. The name suggests the opposite. Always read the data dictionary.

**2. The file has a UTF-8 BOM.**
The first three bytes are `ef bb bf`, an invisible marker left by Excel. It
silently attaches to the first column name, so `df['ID']` raises `KeyError`
while `ID` looks perfectly present. Fix:

```python
df = pd.read_csv("Train.csv", encoding="utf-8-sig")
```

We have to do this because otherwise we would be getting ID errors: 
What we see printed in the terminal:
['ID', 'Name', 'Age']

What Pandas actually sees in memory:
['\ufeffID', 'Name', 'Age']

**3. Warehouse blocks are A, B, C, D, F.**
There is no block E. Do not assume a contiguous range — read the actual values.

## Main finding: the labels are rule-generated

Baseline late rate: **59.7%** (6,563 of 10,999).

Two deterministic rules explain the entire signal:

| Rule | n | Late rate |
|---|---|---|
| `Discount_offered > 10` | 2,647 | 100.00% |
| `Weight_in_gms` in [2000, 4000] | 1,792 | 99.83% |
| Either rule true | 2,919 (26.5%) | 99.90% (3 exceptions) |
| Neither rule true | 8,080 (73.5%) | 45.14% |

The discount cliff is abrupt. Late rate hovers at 44-49% for every discount
value from 1 to 10, then jumps to exactly 100% at 11 and stays there:

```
discount  9  ->  44.3% late   (n=845)
discount 10  ->  46.6% late   (n=860)
discount 11  -> 100.0% late   (n=46)    <- cliff
discount 12  -> 100.0% late   (n=72)
```

Every other column is noise:

| Column | Spread in late rate |
|---|---|
| Discount_offered | 54.1 pts |
| Weight_in_gms | 43.1 pts (non-monotonic) |
| Customer_care_calls | 11.3 pts |
| Prior_purchases | 9.6 pts |
| Cost_of_the_Product | 8.8 pts |
| Product_importance | 5.9 pts |
| Customer_rating | 1.9 pts |
| Warehouse_block | 1.6 pts |
| Mode_of_Shipment | 1.4 pts |
| Gender | 0.5 pts |

The weight effect is non-monotonic, which is the tell that it is a hand-written
band rather than a physical effect. Heavy packages are *safer* than medium ones:

```
1000-2000g  ->  50.7% late
2000-4000g  ->  99.8% late   <- the band
4000-6000g  ->  43.1% late
```

## What this means

This is **label leakage**: the outcome was computed from the inputs by a rule,
not observed in the world. Any model trained here will score near-perfectly and
then fail completely on real data, because the rule does not exist outside this
file.

Consequences for the project:

1. Do not present accuracy numbers from this dataset as if they were real
   predictive performance.
2. Prediction ceiling is only **66.8%** correct, versus 59.7% for blindly
   guessing "late" every time — so pure accuracy is a poor score for a game.
   Hence the token-economy scoring in `DESIGN.md` section 4.
3. The leak is the game's central puzzle rather than a problem to hide.

## Open questions for Phase 2

1. Why is the late rate inverse to `Customer_care_calls` (63% -> 52%)? More
   complaints should mean worse service, not better.
2. Are the 3 rule-breaking orders data entry noise, or a third rule?
3. Is the coin-flip zone genuinely 50/50, or is there a weak signal left in it?
4. Does `Cost_of_the_Product` still matter once discount is controlled for?
5. Block F has exactly 3,666 rows, twice every other block's 1,833. Why?

---

## Gotcha 4: the file is ordered (found during game balancing)

`Train.csv` is **not** in random order. Doomed orders are concentrated at the
top of the file:

```
rows     0- 3000 : doomed ~93%   late 100.0%
rows  3000- 4000 : doomed  12.5%  late  51.5%
rows  4000-10999 : doomed  ~0.1%  late  ~43%
```

2,917 of the 2,919 doomed orders live in the first 8,000 rows. Only `ID` is
monotonic, so this is not a simple sort on any one column — but the ordering
is unmistakable.

### Why it mattered

The game design uses a train/test split: an 8,000-row queryable archive and a
2,999-row live pool. Splitting sequentially gave a live pool containing **2
doomed orders out of 2,999 (0.1%)**. Balance simulation showed expert play
scoring exactly the same as knowing nothing — a $0 skill gap. The game was
unwinnable by skill and it took a simulation to notice.

### The fix

Shuffle before splitting, with a fixed seed for reproducibility:

```python
rng = np.random.default_rng(42)
shuffled = df.iloc[rng.permutation(len(df))].reset_index(drop=True)
archive, live = shuffled.iloc[:8000], shuffled.iloc[8000:]
```

Live pool afterwards: 27.2% doomed, matching the file overall.

### The general lesson

Never split ordered data sequentially. If rows are sorted by anything —
time, category, or an undocumented generation process like this one — a
sequential split produces a test set drawn from a different distribution than
the training set. Results then look fine in development and collapse in
production. Always check, always shuffle, always fix the seed.

Phase 4 must include a test asserting the live pool's doomed rate is within a
few points of the overall rate. That test would have caught this immediately.
