# Foci Workbench

`foci_workbench.html` answers one question:

> Cells in a well fall into two populations, defined by a marker. Each well got
> a different concentration of a drug. **Do the two populations differ in DNA
> damage foci, and does that difference change with the dose?**

Double-click the file, drop in the pipeline's `per_nucleus_measurements.csv.gz`
from a run with `--foci-channel`, and everything below appears. Nothing is
uploaded anywhere; the browser reads the file locally, plain or gzipped, at
full-plate size.

---

## Five steps

**1. Data.** Pick the foci channel (if you counted more than one) and the plate.
*Cells* lets you restrict the analysis to G1 or G2/M nuclei by DNA content (see
[Cell cycle](#cell-cycle-the-confounder-to-check)). Dead and dying nuclei
(bright DAPI, if you ran `--filter-dead --dead-action flag`) are left out by
default.

**Cell filter** (the first card on the right). The same QC windows as the gating
tool: nuclear stain intensity, nuclear area, eccentricity, solidity and
perimeter, whichever the table has. Switch a window on and it starts at the
central 98% of the plate's nuclei; drag either edge or type a value. Use it to
drop debris (tiny, dim), clumps and segmentation errors (large, irregular, low
solidity) and dying nuclei (small, very bright). The histograms always show
every nucleus on the plate, so a distribution doesn't shift while you drag its
own window.

The **foci channel's mean intensity** is offered as a window too, labelled
*pan-nuclear*. Nuclei with pan-nuclear γ-H2AX are uniformly bright, and their
texture gets counted as dozens of "foci". Put the upper edge of that window
below the bright tail to leave them out of the foci counts. Because the share of
pan-nuclear cells usually rises with dose (it marks the most damaged cells),
switching that window on also adds the readout **% pan-nuclear nuclei**, so
those cells are reported rather than silently dropped.

The filter is applied *first*: the population gate, the
autofluorescence correction, every plot and every statistic see only the
nuclei that pass, and the export records the windows used.

**2. Populations.** Choose the marker channel. Its intensity histogram appears
with a threshold, placed automatically (Otsu, on a log scale). Drag the line or
type a value. Nuclei above it form the second population. Rename the two
populations: the names are used in every plot, table and sentence. If the
pipeline already made the call, choose *“pipeline's positive call”* instead.

**3. Concentrations.** If the concentration is stored in OMERO as a key-value
pair, it is in the file as a `kv_` column and is picked up automatically. If
not, choose *“type the concentrations below”* and enter one line per well:

```
A01  DMSO
A02  0,01
A03  0.03 µM
```

`0`, `DMSO`, `vehicle`, `control` mean the control; decimal commas and units
are fine; `A1` and `A01` are the same well; a well with no value is left out.
Press **Apply**. Lines that can't be read are listed, not silently dropped.

**4. Readout.** What to compare per well: mean foci per nucleus, % of nuclei
with ≥ k foci, median, 90th percentile, mean foci in nuclei that have any, and
foci area fraction / integrated intensity if present. A population with fewer
than the minimum number of cells in a well (default 20) is not used for that
well.

**5. Read the results**, top to bottom.

---

## The plots

| Plot | What it shows |
|---|---|
| **Foci across the dose series** | the readout per well (dots) and per concentration (line), one colour per population |
| **Difference per well** | population 2 minus population 1, *within each well*, against concentration. This is the comparison the tests make. |
| **% with ≥ k foci** | for each population, one curve per concentration: the share of nuclei with at least k foci, for every k. A curve moving right means more cells with more foci. It shows the *whole distribution*, which a mean or median hides. |
| **Distribution bins** | the same information as stacked bars (0 / 1–2 / 3–5 / 6–10 / 11+ foci, editable) |
| **SuperPlot** | every nucleus as a faint dot, every well mean as a large dot. The large dots are the replicates; the cloud shows how heterogeneous the cells are. |

The distribution plots pool the nuclei of all wells at a concentration. That is
right for *looking* at the shape. The tests never pool cells this way.

---

## The statistics

### Why the well is the replicate

A plate gives you thousands of nuclei but only a few dozen wells. Nuclei in one
well share everything that varies between wells (seeding, drug pipetting, edge
effects, focus), so they are not independent. Test 5,000 nuclei as if they
were, and a difference of no biological consequence comes out at p < 10⁻¹⁰.
Every test here therefore lets the **wells** set the uncertainty.

### Population difference, paired by well

Both populations live in the same well, so the natural comparison is the
**difference within each well**. That cancels everything the two populations
share in that well. With *n* wells you get *n* paired differences, tested with:

- **paired t-test**, with the mean difference and its 95% CI;
- **Wilcoxon signed-rank**, which doesn't assume the differences are normally
  distributed (exact p for up to 50 wells, ties handled exactly);
- the **ratio** population 2 / population 1 as a geometric mean, when all
  readouts are positive.

This works **even with one well per concentration**: every well is one pair.

The table below it repeats the comparison **per concentration**, but only
where there are at least 3 wells at that dose, with Holm correction for testing
several doses.

### Dose trend

**Spearman correlation** across wells between concentration and the readout,
for each population and for the within-well difference. It asks “does this rise
(or fall) steadily with dose?” without assuming a curve shape. The control
ranks lowest. A significant trend in the *difference* means the two populations
respond differently to the drug.

### Cell-level model

The well-level tests summarise each well by one number. The model uses every
nucleus while still respecting the wells:

- **negative binomial regression** on the foci count of each nucleus (the
  standard model for over-dispersed counts: most nuclei with few foci, a long
  tail with many), or
- **logistic regression** on whether a nucleus has ≥ k foci,

with **population × concentration**, and **standard errors clustered by
well**. CIs and p-values use a t distribution with (wells − 1) degrees of
freedom. The results are given as ratios:

| Row | Meaning |
|---|---|
| population 2 vs 1, in control wells | the population difference without drug |
| population 2 vs 1, at the top dose | the difference at the highest concentration |
| concentration effect in each population | how foci change per 10× concentration (or per dose step) |
| **difference in dose response** | the interaction. × 1 means both populations respond alike; × 1.25 means foci rise 25% faster per 10× in population 2. **This is usually the answer to the question.** |

Concentration enters either as **log₁₀ concentration**, with the control placed
one dose step below the lowest dose (a trend on the log scale), or as **dose
steps** (control = 0, each next dose +1). Try both: a conclusion that holds
under both does not depend on how the dose axis was drawn.

With fewer than about 10 wells, clustered standard errors are approximate;
then rely on the paired well-level tests. The page says so when it applies.

**Validation.** Every statistic was checked against `scipy` and `statsmodels`
on the same cells: paired t, Spearman, Wilcoxon, the negative binomial
dispersion, coefficients and clustered standard errors (to 7 significant
digits), the logistic model, and the DNA-adjusted model.

---

## Autofluorescent compounds (doxorubicin and friends)

Doxorubicin, daunorubicin, epirubicin and mitoxantrone fluoresce, and the
anthracyclines accumulate in the nucleus. Their signal adds to the marker
channel and grows with dose, so marker-**negative** nuclei drift up across a
fixed threshold. At high doses the "positive" population then fills with
negative cells, and a real difference between the populations can vanish
entirely. On a simulated doxorubicin plate, a true interaction of × 1.28
came out as × 1.01 without correction.

**Spotting it.** The card *Marker across the dose series* shows the marker
histogram at every concentration (a ridgeline), with the threshold. If the
negative peak creeps right with dose, it's autofluorescence. The page also
warns when the negative peak is more than 1.5× brighter at the top dose than in
the control.

**Correcting it.** Under *Populations → Autofluorescence correction*:

| Option | Model | Use when |
|---|---|---|
| **per-cell compensation from a reporter channel** | `marker − k × (reporter − baseline)`, per nucleus | **best**, whenever one of your channels sees the compound but not the marker |
| negative-peak shift, fitted across concentrations | subtracts the dose-dependent shift of the negative peak, fitted as a non-decreasing curve | no reporter channel; moderate autofluorescence |
| negative-peak shift, per well | the same, each well's own shift | replicate-free designs with many cells per well |

All three are additive in raw intensity, because that is how fluorescence
adds up. The threshold is placed again automatically after switching; check it.

**Why the reporter wins.** Drug uptake differs from cell to cell. At high
doses the extra signal varies by tens of percent between nuclei in the *same*
well, which can be more than the gap between negative and positive cells. A
shift per well or per dose moves the whole distribution but cannot undo that
spread; only a per-cell measure of each nucleus's drug content can. In the
simulation, cells correctly classified at 10 µM were 31% uncorrected, 44% with
the fitted shift, and **99.5% with reporter compensation**.

**How k is fitted.** Cells from all wells are binned by reporter intensity. In
each bin a low quantile of the marker is taken, which describes negative cells
only, and k is the robust (Theil–Sen) slope of that quantile against the
reporter. The right-hand plot shows the bins and the fitted line; it should be
straight. On the simulation, k came out at 0.453, against a true value of 0.455.

**Choosing a reporter channel.** Doxorubicin excites around 480 nm and emits
around 590 nm, so it is brightest in orange-red channels (TRITC/Cy3) and weaker
in GFP/FITC. Any channel that shows the drug and *not* the marker works. If you
have none, acquire one extra channel with no stain in it: an empty TRITC
channel is a cheap, near-perfect autofluorescence reporter. For future
experiments, a far-red marker sidesteps the problem.

**Assumptions to keep in mind.** The peak-shift options assume the negative
cells' *true* marker level does not change with dose; if the drug genuinely
changes marker expression, they will remove real biology. The reporter option
assumes the reporter channel contains no marker signal. Wells with
marker-negative cells only, treated with the drug, are the cleanest check of
either correction.

---

## Cell cycle: the confounder to check

Nuclei in S and G2 carry more DNA, and γ-H2AX also marks replication stress, so
G2 nuclei typically have more foci without having more damage. If the drug
arrests cells in G2, or if one population cycles more than the other, the foci
difference can come from the cell cycle rather than from DNA damage.

Two ways to check, using the DNA content (integrated nuclear stain) already in
the table:

1. **Cells → G1 only.** Re-run the analysis on 2N nuclei only. The 2N and 4N
   peaks are found automatically and shown in the sidebar; check they sit on
   the peaks.
2. **Adjust for DNA content** in the model, which adds DNA content as a
   covariate (the effect per doubling of DNA is reported).

If the conclusion survives both, the cell cycle is not the explanation.

---

## Exporting

**Export results** writes a CSV with one row per well: concentration, cells per
population, and every readout for both populations. The test results, the gate
and the settings follow as `#` comment lines, so the file documents itself.
Read it in Python with `pd.read_csv(path, comment="#")`.

---

## Honest limits

- **One compound per analysis.** For several compounds, untick the other wells
  or give only one compound's wells a concentration.
- **Foci counts at 20x saturate** in heavily damaged nuclei, because touching
  foci merge. If the top doses plateau, look at *foci area fraction* or
  *integrated foci intensity*, which keep rising.
- **The gate matters.** Check the marker histogram has two clear peaks and the
  threshold sits in the valley. The populations' percentages are shown under
  the histogram, and the page warns when one population is very small.
- **One well per concentration** means no replication at any single dose. The
  population comparison and the trend are still valid, but “is dose X different
  from the control?” cannot be tested. Plan replicate wells if you need that.
