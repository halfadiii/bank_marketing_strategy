# Bank marketing: who is worth calling

**45,211 real sales calls from a Portuguese bank. Almost nobody says yes. Six
notebooks clean the calls, put them in a small database, test what goes with a
yes, and fit three models to rank who to ring first.**

The bank was selling a term deposit: a savings account where the money is locked
away for a fixed time. Calls cost money, and of the 43,193 calls left after
cleaning, 5,021 ended in a yes. That is 11.6%. So the question underneath is a
plain one: who do you call first?

The best model here scores **0.916 ROC AUC**. Half of that comes from a column
you only know once the call is over. Take it out and the score is **0.765**.
Both numbers are in the notebooks, and the second is the one a bank could use.

**Live dashboard:** [adityaaryan.in/dashboard/bank-marketing](https://adityaaryan.in/dashboard/bank-marketing),
the same cleaned calls, filtered in the browser.

---

## The notebooks, in order

Each one starts from what the one before it saved.

| Notebook | What it does | What it leaves behind |
| --- | --- | --- |
| `part-1_part-2.ipynb` | Looks at every column and cleans the data | `Final tables/cleaned-data.csv` |
| `part-3-normalization.ipynb` | Gives each call an id and splits the data into two tables | `Final tables/main-table.csv` and `p-outcome-table.csv` |
| `part-3-SQL.ipynb` | Builds the SQLite database with its keys, and one saved query that joins the tables | `mydatabase.db` |
| `part-4.ipynb` | Charts: age, balance, job, and how the numbers move together | |
| `part-5.ipynb` | The tests and the three models | |
| `part-6.ipynb` | A dashboard in Dash and Plotly | |

The first five are saved with their results, so everything below can be read
here without running anything.

---

## Cleaning, and what each step costs

| Step | Rows after | Why, and what it costs |
| --- | --- | --- |
| Load `bank-full.csv` | 45,211 | |
| Drop `campaign` and `day` | 45,211 | Both are only weakly related to the answer (correlations of -0.07 and -0.03). So are `age` and `balance`, which stay. A simplification, not a finding. |
| Remove unknown job | 44,923 | 288 rows, under 1%. |
| Remove unknown education | 43,193 | 1,730 rows. This assumes those people are like everyone else. They may not be. |
| Reassign unknown contact method | 43,193 | 12,286 calls had none recorded, too many to throw away. They are shared between mobile and landline in the proportion of the known ones. The totals stay honest. Each single assignment is a guess. |
| Fold unknown previous outcome into `other` | 43,193 | Both mean there is no useful history for this person. |

---

## The database

The cleaned calls go into SQLite as two tables.

- **`main_table`**: one row per call, with an `id`. The data has no key of its
  own (one row is an exact copy of another), so each call is given one.
- **`p_outcome_table`**: what is known about that person's previous campaign,
  under the same `id`. It has a row only for the 7,912 calls where there was one.

Why split at all: 35,281 calls, four in five, went to people who had never been
contacted before. The file writes that as three placeholders on every one of
those rows (`pdays` -1, `previous` 0, outcome unknown). That is one fact stored
three times. In its own table it is stored once, by there being no row.

What the split is not: the first idea was a third-normal-form fix, moving the
previous outcome into a lookup table keyed on `pdays` and `previous`. The
notebook tests that idea before using it, and it fails. 903 of the 2,404 pairs
appear with more than one outcome, so the outcome does not follow from the other
two and the notebook does not build that table.

The two tables are joined in one place, a view called `calls`. The charts, the
models and the dashboard all read from it. The SQL notebook ends by checking
that the view gives back the cleaned data exactly.

---

## What the data says

Before any model, part 5 asks whether each pattern could be chance.

| Question | Test | Statistic | p |
| --- | --- | --- | --- |
| Previous campaign outcome and saying yes | Chi-squared | 4,027.7 | too small to print |
| Housing loan and saying yes | Chi-squared | 825.3 | 1.7e-181 |
| Job and saying yes | Chi-squared | 772.5 | 1.7e-159 |
| Education and saying yes | Chi-squared | 233.4 | 2.1e-51 |
| Balance, previous success against failure | Welch t-test | 4.29 | 1.9e-05 |

With 43,193 rows almost anything passes a test. These say a pattern is real, not
that it is large. The sizes:

- **What happened last time matters most.** Of the 1,424 people whose previous
  campaign ended in a yes, 64.4% said yes again. After a failure it was 12.5%,
  and for everyone else 9.5%.
- **A housing loan halves it.** 7.7% of people with one said yes, against 16.6%
  without. A marketing team can use that the same afternoon, with no model at
  all.

---

## Three models

Trained on 80% of the calls and tested on the 8,639 they never saw. The split is
stratified, so both parts hold the same 11.6% of yes.

| Model | Accuracy | Precision | Recall | F1 | ROC AUC |
| --- | --- | --- | --- | --- | --- |
| Always say no | 0.8838 | | | | |
| Logistic regression | 0.9027 | 0.6468 | 0.3576 | 0.4606 | 0.9099 |
| Decision tree, unpruned | 0.8713 | 0.4501 | 0.4851 | 0.4669 | 0.7036 |
| Gradient boosting | 0.9064 | 0.6507 | 0.4193 | 0.5100 | 0.9164 |

Read the accuracy column first, because it is the trap. A model that says no to
everyone scores 0.884 and beats the decision tree. Accuracy cannot rank these.
ROC AUC can: it is the chance that the model puts someone who said yes above
someone who said no.

---

## The flaw: the best column is the answer in disguise

Gradient boosting leans on `duration`, the length of the call, for half of its
decisions. You only know how long a call lasted once it is over, and a call of
zero seconds was never a yes. So a model that uses it cannot tell you who to
ring. The people who published the data say so themselves: include it for
benchmarks, discard it for any realistic model.

So part 5 fits the same model on the same rows without that one column.

| | With duration | Without it | Change |
| --- | --- | --- | --- |
| ROC AUC | 0.9164 | 0.7645 | -0.1518 |
| F1 | 0.5100 | 0.3133 | -0.1967 |
| Recall | 0.4193 | 0.2082 | -0.2112 |
| Precision | 0.6507 | 0.6333 | -0.0174 |
| Accuracy | 0.9064 | 0.8940 | -0.0124 |

Fifteen points of ROC AUC go. Accuracy moves by one. Judged on accuracy, nobody
would have noticed that half the model had just disappeared.

0.916 is a benchmark number. 0.765 is the one a bank could use.

---

## Run it

```bash
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute --inplace part-1_part-2.ipynb part-3-normalization.ipynb part-3-SQL.ipynb part-4.ipynb part-5.ipynb
```

That rebuilds everything from `bank-full.csv` in under a minute. Or open the
notebooks in Jupyter and run each one top to bottom, in that order.

The dashboard needs the database, so it goes last. Open `part-6.ipynb` and run
all of it, and it serves the dashboard from your own machine: five filters (job,
marital status, education, month, balance) and four charts.

---

## State of things, honestly

**This was repaired in October 2026.** The first version, from January 2025, was
broken in the middle, and I would rather say so than have you find it in the
history. The cleaning notebook never saved its result, so the database was built
from the raw file. The second table was the lookup described above, the one the
data does not support. And the two tables were joined by row number, which
returned 3,660 of the 45,211 calls, nearly all with the wrong previous outcome
attached. The models and the dashboard ran on those rows, and reported accuracy
only. Each step now prints the check that proves it, and every number on this
page is one the notebooks print.

**The decision tree is a straw man.** It is unpruned, so it memorised the calls
it learned from. A depth limit would make it a fair comparison.

**Nothing corrects for the imbalance.** 88 no to 12 yes, with no class weights
and no resampling. That is why recall sits at 0.42: the best model misses more
than half of the people who would have said yes.

**One split.** An 80/20 split gives one number and no sense of how far it would
move on a different split. Cross-validation would say.

**The unknowns were deleted or guessed.** See the cleaning table. Keeping
"unknown" as a category of its own may have been the better call.

---

## The data

[UCI Bank Marketing](https://archive.ics.uci.edu/dataset/222/bank+marketing),
licensed CC BY 4.0. Moro, S., Cortez, P. and Rita, P. (2014). A data-driven
approach to predict the success of bank telemarketing. *Decision Support
Systems*.
