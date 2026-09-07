# Flat Hunter

A weighted multi-criteria scoring tool for comparing rental flats.

## Motivation

I wanted to build this tool as I came across this problem when I was looking for a rental flat for my second year housing. The flats were shared in a Whatsapp group chat, each as a separate link. There wasn't an easy way to see the flats side by side and we had to open each link individually to compare. Hence, it was inefficient and easy to lose track as we were comparing based on memory across all the open tabs.

Looking for flats is not a single-variable decision - it is a tradeoff across several things at once (e.g. price, commute, size, bills included, furnished etc.). Furthermore, different people will weigh these critieria differently, for example someone might trade a longer commute for a cheaper flat but someone else might not. A simple sort-by-price or sort-by-size list cannot capture that personal tradeoff.

Therefore, I built this tool to let the user weigh the importance of each criterion themselves, and produce one transparent, comparable score per flat — not just a single number, but a full breakdown of why a flat ranked where it did.


## How the scoring model works

Every criterion is scored on a scale of 0-10, regardless of what it originally measures (£/week, minutes, sqm, yes/no). The user assigns a raw importance weight to each criterion — any positive number, reflecting how much that criterion matters to them relative to the others. Weights are then normalised to sum to 1, and the total score is computed by summing each criterion's score multiplied by its normalised weight. This keeps every flat's total score on the same 0-10 scale, regardless of how many criteria are used or what raw numbers were typed in, so the number remains interpretable, not just useful for ranking.

Two normalisation approaches were used: anchored scoring for price and commute, and relative min-max scoring for size. Anchored scoring was used when the user has a personal, absolute reference point independent of what flats happen to be in the list, such as a budget or a comfortable commute time. It uses two anchors: below the lower anchor (e.g. lower budget), the score is clamped at 10; between the two anchors, it declines linearly from 10 down to 5; and past the upper anchor (e.g. max budget), it keeps declining below 5, floored at 0. Relative min-max scoring was used when there is no natural absolute anchor, so the criterion is instead scored relative to the other flats currently in the list. This split exists because forcing one universal method would either lose the "compared to my own budget or tolerance" signal (if scoring were relative-only), or have no sensible reference point for a criterion like size (if scoring were anchored-only).

For criteria such as bills-included and furnished, where the answer is either yes or no, there is no numeric scale to normalise, so the model uses a simple lookup instead: the user's preferred value scores 10, the other scores 0. The preferred value itself is set by the user, not assumed (e.g. furnished is not hardcoded as always better, since not everyone will prefer it).

## Hard filters

This is an optional pre-scoring filter step, currently used to filter out flats below a minimum bedroom count. It runs before weighted scoring, not as part of it, because true must-haves are pass/fail — not something that should just be scored badly. For example, if two people are looking to rent together, the flat has to have at least two bedrooms; that's fundamentally different from something like an over-budget flat, which still gets scored, just poorly. This step exists to catch listings that slip through without meeting every compulsory requirement — even though the tool assumes flats are roughly pre-qualified via rental sites, that assumption doesn't always hold in practice.

## Score breakdown

Every score is broken down per criterion, showing the raw value, normalised score (0-10), normalised weight, and contribution (weight × score) — the four fields built into breakdown. Contributions sum exactly to the total score, so the breakdown isn't just descriptive, it fully accounts for the number itself. This is done deliberately for explainability: rather than relying on any learned or black-box component, every score is the result of a plain, traceable calculation — I want the user to be able to see exactly why a flat ranked where it did, not just trust a single opaque number.

## Project structure

- **data.py**: sample data and user configuration
- **normalise.py**: includes all the normalisation functions (anchored scoring, min-max, boolean lookup, weight normalisation)
- **scoring.py**: combines a flat's raw values with the normalise functions into a full score breakdown and total score
- **filters.py**: the hard-filter step
- **display.py**: formats and prints the comparison table
- **main.py**: wires everything together and runs the program
- **test_normalise.py, test_scoring.py, test_filters.py**: pytest test suites, one per module

## How to run it

Clone the repo, then set up a virtual environment and install dependencies:

```bash
git clone https://github.com/janicejclee/flat-hunter.git
cd flat-hunter
python -m venv venv
source venv/Scripts/activate   # Git Bash on Windows; use venv/bin/activate on macOS/Linux
pip install -r requirements.txt
```

Run the tool:

```bash
python main.py
```

Run the test suite:

```bash
pytest
```

Flats and configuration (budget, commute tolerance, preferences, weights, minimum bedrooms) are currently set directly in `data.py` — there's no user-facing input yet, since that's planned for the upcoming Flask phase.

## Sample output

Example output for the sample flats in `data.py`:

```
                Flat A         Flat C         Flat D         Flat E         
total score     7.27           7.88           8.00           5.79           

commute (weight: 0.3125)
score           8.33           8.00           6.67           3.33           
commute         10             12             20             40             
contribution    2.60           2.50           2.08           1.04           

price (weight: 0.25)
score           5.00           4.00           7.50           9.00           
price           400            420            350            320            
contribution    1.25           1.00           1.88           2.25           

furnished (weight: 0.25)
score           10.00          10.00          10.00          10.00          
furnished       1              1              1              1              
contribution    2.50           2.50           2.50           2.50           

size (weight: 0.125)
score           7.33           10.00          7.33           0.00           
size            588            620            588            500            
contribution    0.92           1.25           0.92           0.00           

bills_included (weight: 0.0625)
score           0.00           10.00          10.00          0.00           
bills_included  1              0              0              1              
contribution    0.00           0.62           0.62           0.00           
```

(Flat B is filtered out here — it doesn't meet the minimum bedroom requirement.)

## Design decisions

**Over-budget flats aren't hard-excluded**
A flat above the max budget still gets scored (just poorly, continuing the same linear decline below 5, floored at 0) rather than being rejected entirely. This is because a flat slightly over budget but excellent in other criteria might still be worth surfacing; hard exclusion would lose the ability to show "here's what a bit more money gets you."

**Tie-handling in min-max scoring**
If every flat ties on a relative-scored criterion (e.g. all the same size), they all get a neutral score of 5, rather than an arbitrary tiebreak. The reasoning behind this is that if there is no variation across the list, there is no real information to differentiate flats on, so inventing a distinction would be misleading.

**Rejected: rank-based normalisation for size**
Scoring size by rank was considered (1st biggest, 2nd biggest, etc.) instead of min-max, to avoid min-max's sensitivity to outliers. This was rejected because rank-based scoring leads to large score gaps between flats that are actually very similar in size, which is the more common real-world case for this use case.
 
## Known limitations / what's next

**Flask/SQL phase**
SQL will replace the hardcoded flats list in data.py with persistent storage, so the flats stay saved across sessions instead of being retyped by hand. Flask will replace main.py's terminal entry point with a real web front-end as a form for entering flat data and setting weights/preferences, with the ranked table rendered as a web page instead of printed to a terminal.

**Real listing scraping**
This tool would eventually be able to pull flats directly from rental sites instead of manual entry, using the URLs provided by the user. The tool would also compute price-per-person from a listing's total rent and number of tenants/bedrooms, then compare that against the user's own stated personal budget.

**Single-destination commute**
v1 assumes one dominant destination (uni). Future renditions plan to support multi-destination commute (work, gym, supermarket etc.), but this would add real complexity to weight input and the breakdown.

**Min-max sensitivity to outliers**
Right now, a single extreme value on a relative-scored criterion (e.g. size) compresses the resolution among the rest of the list. This is deliberately accepted for v1 given low real-world likelihood, but a possible v2 fix is percentile-rank or winsorizing.

**Possible future AI element**
This is a plausible extension for parsing messy real-world listing text into structured fields. This was explicitly kept out of the core scoring math itself to preserve explainability.

