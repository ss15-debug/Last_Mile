# Last Mile

A dispatch game where you have to figure out why the packages are late.

Every shift, 20 real shipping orders cross your desk. Five of them you can
expedite. Nobody tells you which ones are doomed — you have to work that out
from the evidence.

> **Status:** in development. Phase 1 of 5 (design complete, game not yet built).
> This is my first programming project.

---

## The idea

Built on 10,999 real-world-shaped e-commerce shipping records from Kaggle.
While exploring the data I found that the delivery outcomes are driven by two
hidden rules that the dataset never documents. Rather than hide that, the game
is built around it: **discovering the rules is how you win.**

The game ships with a Notebook panel where you can query historical orders —
group by any column, see the late rates, form a theory, test it. That is the
actual skill the game is about.

See [DESIGN.md](DESIGN.md) for the full design and
[docs/DATA_NOTES.md](docs/DATA_NOTES.md) for what the data analysis turned up.

## Data

[E-Commerce Shipping Data](https://www.kaggle.com/datasets/prachi13/customer-analytics)
— 10,999 orders, 12 columns, no missing values.

Three things worth knowing if you use this dataset yourself:

- `Reached.on.Time_Y.N` is inverted — `1` means late
- the file carries a UTF-8 BOM, so read it with `encoding="utf-8-sig"`
- warehouse blocks are A, B, C, D, F — there is no E

Full write-up in [docs/DATA_NOTES.md](docs/DATA_NOTES.md).

## Running it

Requires Python 3.12+.

```bash
git clone https://github.com/ss15-debug/Last_Mile.git
cd Last_Mile
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python -m game                   # not yet implemented
```

## Project layout

```
game/
  logic.py       rules and scoring — no UI, fully testable
  data.py        loads the dataset, owns the train/test split
  ui.py          drawing and input — contains no rules
analysis.py      data exploration, generates the charts in docs/
tests/           pytest suite
docs/            design notes, findings, wireframes
```

The hard rule: `logic.py` never imports the UI, and `ui.py` never contains
rules. Logic runs headless with no window open, which is what makes it
testable.

## Roadmap

- [x] **v0.1** — project setup, dataset in version control
- [ ] **v0.2** — design document, wireframes, data notes
- [ ] **v0.3** — NumPy analysis complete, probability tables exported
- [ ] **v0.5** — playable in the terminal
- [ ] **v0.8** — graphical UI with art
- [ ] **v0.9** — pytest suite and CI
- [ ] **v1.0** — released

## Why this exists

I am learning to program, and I wanted a first project that covered the whole
arc rather than just the fun part: planning before building, real data analysis,
tests, version control, and actually shipping it. Notes on what broke and what
I learned are in [docs/](docs/).

## License

MIT — see [LICENSE](LICENSE).
Dataset is used under its own Kaggle license and is not mine.
