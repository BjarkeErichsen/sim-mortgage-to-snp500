# How the simulation calculates

Context: a Danish real-estate **ApS** raises a *realkreditlån* against properties it already
owns and invests the proceeds in the S&P 500. The simulation asks what the net equity of
that trade looks like after 30 years, across many possible return sequences.

All amounts are in DKK.

---

## 1. One-time setup

**Return series** (`histretSP.xls`, Damodaran, "Returns by year", 1928–2025):

1. Read the S&P 500 **total** return (includes dividends) and US CPI inflation for each year.
2. Convert to a real return with Fisher: `real = (1 + nominal) / (1 + inflation) - 1`.
3. What happens next depends on the switch **`USE_INFLATION_ADJUSTED_RETURNS`** (cell 1):

   **`True`** — add back a flat 2%: `adjusted = real + 0.02`. This discards the historical
   inflation *path* (1946's 18%, the 1970s) and replaces it with a constant, so no path is
   distorted by an inflation regime that won't repeat. Returns are then nominal, in
   "2%-inflation kroner", and the deflator is a deterministic `1.02^t` on every path.

   **`False`** — sample the **nominal** return instead, paired with that year's **realised**
   inflation. Both are drawn from the same calendar year (the sampler collects an *index*
   path, so they cannot decouple). The simulation runs entirely in nominal kroner and each
   path carries its own deflator.

   The difference matters because the mortgage is a fixed **nominal** liability. Mode `True`
   erodes it at exactly 2%/yr, every path. Mode `False` lets inflation do what it actually
   did — which is the only way to price the inflation hedge the loan contains. On the default
   scenario this is worth roughly **+2.9M in today's DKK** (median 11.0M vs 14.0M), because
   mode `True` keeps the real-return damage inflation does to equities while withholding the
   relief it gives the borrower.

**Loan** (`loan_schedule`), a 30-year fixed-rate annuity:

4. We owe the **face** amount (10,000,000) but receive `face × kurs` = 9,500,000. The
   500,000 *kurstab* is treated as an immediate cost — it is never recovered or deducted.
5. The yearly payment is an annuity on the **full face**: `P·r·(1+r)^n / ((1+r)^n − 1)`
   = 688,054/yr, constant for all 30 years.
6. Each year is split: `interest = outstanding balance × 5.5%`, `principal = payment − interest`.
   Interest therefore starts high (550,000) and falls as the balance amortises.

**Starting position:** the portfolio is the 9,500,000 actually received. Debt is 10,000,000.

---

## 2. Each year, in order

For year `t`, starting from portfolio value `V_start`:

| Step | Calculation |
|---|---|
| 1 | Sample this year's return `r` from the adjusted real-return series |
| 2 | Apply it: `V_after = V_start × (1 + r)` |
| 3 | Measure the gain: `value_change = V_after − V_start` |
| 4 | Look up `interest` for year `t` from the amortisation schedule |
| 5 | `taxable_income = value_change − interest − loss_carryforward` |
| 6 | If positive → `tax = 22% × taxable_income`, and reset `loss_carryforward = 0`<br>If negative → `tax = 0`, and `loss_carryforward = −taxable_income` |
| 7 | `V_end = V_after − tax − payment − expenditure`, floored at 0 |
| 8 | `net_gain = V_end − remaining debt` |

`V_end` becomes next year's `V_start`.

**Why step 3 measures the gain before the withdrawal.** Danish *lagerprincippet* taxes the
year's change in value, and sales proceeds count toward that change. Selling shares to make
the mortgage payment is therefore **not** a way to shelter a gain — the tax is owed on the
full `value_change` even though most of it leaves the portfolio in step 7. Measuring the gain
against anything other than the value the portfolio actually started the year with (for
example against a running high-water mark set before withdrawals) lets withdrawn money stay
in the tax basis and makes the tax collapse toward zero.

**Why losses carry forward instead of being refunded.** A loss year, or a year where the
interest deduction exceeds the gain, produces no refund. It becomes an *underskud* carried
forward indefinitely and offsets future taxable income (step 5). Interest is fully deductible
for a company, so a bad year still "banks" its deduction rather than wasting it.

---

## 3. Monte Carlo and outputs

7. Pick a random start year, then walk **forward** through actual history, so sequences keep
   their real ordering (crashes, recoveries, mean reversion). If a path runs past the end of
   the data it jumps to a new random start year and continues.
8. Repeat for 500 paths; take the 2.5th / 50th / 97.5th percentile **per year** across paths.
9. The final cell shows the portfolio, the **debt**, and the net gain in *both* nominal kroner
   and today's purchasing power, dividing each path by its own cumulative deflator. The two
   debt curves separating is the inflation hedge made visible.

Those bands are the spread of *outcomes*, not a confidence interval on a mean. They are also
re-samples of only 69 distinct 30-year windows — 98 years of data at a 30-year horizon is
about 3 independent observations, so the tails are far less certain than 500 paths suggests.

**Reading the numbers:** the simulation is always in nominal kroner, so cells 8-10 report
nominal figures. The final cell converts to today's purchasing power for you. Doing it by
hand only works in mode `True`, where the factor is a fixed `1.02^30 = 1.811`; in mode
`False` every path has its own (median 2.61x, range 1.26x-5.10x).

---

## 4. Known simplifications

Each of these pushes the result in the stated direction:

| Not modelled | Effect |
|---|---|
| **Bidragssats** (0.6–1.1% p.a. of balance for erhverv) — the largest omission | too optimistic |
| Fund TER and non-reclaimable US dividend withholding (~0.3–0.5% p.a.) | too optimistic |
| Portfolio floored at 0 while payments continue, so terminal loss cannot go below 0 — a wipeout at year 12 still reports `net_gain = 0` | hides the downside |
| `real + 0.02` instead of `(1 + real) × 1.02 − 1`, in mode `True` only | small, ~0.2pp in a good year |
| ~~Fixed 2% inflation removes the inflation hedge~~ — **now modelled**: set `USE_INFLATION_ADJUSTED_RETURNS = False` | was too pessimistic by ~2.9M |
| The callable bond: if rates rise the loan can be bought back below par, a real option worth most exactly when equities hurt | too pessimistic |
| Unhedged USD/DKK exposure (US returns deflated by US CPI) | understates variance |
| 60% *underskudsbegrænsning* above ~9.8M of income; kurstab deductibility timing | minor, verify with revisor |
| Personal tax on extraction (27%/42% aktieindkomst) — this is a pre-extraction figure | n/a |
