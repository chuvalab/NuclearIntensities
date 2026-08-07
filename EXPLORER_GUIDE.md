# Gating Explorer — a guide for students

`gating_explorer.html` lets you re-analyse a finished experiment by hand:
include or exclude wells, switch markers on and off, and drag the gates to see
how the answer changes. It runs entirely in your web browser. There is nothing
to install, no Python, and **your data never leaves your computer** — the file
is opened locally by the browser, not uploaded anywhere.

---

## Opening it

**If your supervisor ran the pipeline with `--export-explorer`,** you already
have a single file with your data inside it. Double-click
`gating_explorer.html`. That's the whole procedure.

**Otherwise**, double-click `gating_explorer.html` and drag your
`per_nucleus_measurements.csv` onto the window. If your file ends in `.csv.gz`
it will be unzipped automatically, provided you have internet access; if you're
offline, ask for it as plain `.csv` (`--per-nucleus-format csv`).

It handles around a million nuclei. A 96-well plate with 250,000 cells takes
about a second to load, after which everything is instant.

---

## The four things you can change

### 1. Wells — click the plate

The plate map on the left is both the well selector **and** the results
heatmap. Each square is a well, coloured by whatever you choose in the dropdown
underneath (cells per well, % double positive, and so on). Hover for the exact
numbers.

**Click a well to exclude it.** It becomes a dashed outline, and every plot,
percentage and table below updates immediately with that well's cells removed.
Click again to bring it back. "Invert" swaps your selection; "All wells" starts
over.

Use this to drop wells that failed — an obvious edge effect, a dispensing
error, a well where the cell count collapsed. Set the map to **cells per well**
first: bad wells usually announce themselves as a spatial pattern.

### 2. Markers — click to switch off

Under "Markers", clicking a marker greys it out and removes it from the
analysis entirely. This is the important one: with three markers on you get
negative / single / double / triple; switch one off and you get
negative / single / double, recalculated over the two remaining markers, always
summing to 100%.

That is exactly what you want if part of your plate used a different staining
panel — turn off the marker that wasn't stained, and the numbers become correct
for those wells.

### 3. Gates — drag the yellow lines

Each active marker gets a histogram with a yellow line at its threshold. Drag
the line, or type an exact number in the box. Everything recomputes as you
drag.

The histogram is bright where cells count as positive and grey where they
don't, so you can see immediately what a gate is including. Axes are asinh
(log-like, but it handles zero properly), the same as the plots from the
pipeline.

**Where should the line go?** In the valley between two humps. If there's only
one hump, there is no honest threshold, and you should say so rather than
picking a number that splits it. "auto" returns a marker to the threshold the
pipeline chose.

When you open a file produced by the pipeline, the gates start exactly where
the pipeline put them, so your first view matches your report. Anything you
change from there is yours.

### 4. Intensity statistic

`mean` is concentration-like and the usual choice. `median` ignores a few
bright specks of debris. `integrated` is the total per nucleus, so it scales
with nuclear size. Changing this re-derives the automatic thresholds.

---

## Reading the rest of the screen

- **Populations** — two stacked bars per well. The left is composition
  (everything sums to 100%); the right is absolute cell numbers, so bar height
  is the well's cell count. Check both. A well can look excellent by percentage
  and be nearly empty.
- **Biaxial** — the flow-cytometry view, sitting directly beside the gates so you
  can drag on the left and watch the quadrants move on the right without looking
  away. A density plot for every pair of active markers, with the gate crosshair
  and four quadrant percentages that always sum to 100%. **It updates as you
  drag.** Move the GFP gate and the
  GFP × RFP and GFP × Cy5 plots redraw their crosshairs and recount their
  quadrants immediately; switch a marker off and its plots disappear, switch it
  on and they come back.

  Two things to keep in mind. The cell cloud itself does not move when you drag
  a gate — only the crosshair and the percentages do, because you are
  reclassifying the same cells, not changing them. And the top-right quadrant of
  a GFP × RFP plot contains cells that are GFP+RFP+ *whether or not* they are
  also Cy5 positive; for the unambiguous three-marker breakdown, read the
  population bars.
- **Per-well summary** — the same numbers as a table.

---

## Getting your results out

**Export summary** downloads a CSV with one row per **included** well: cell
counts, per-marker positives, and every population as both counts and
percentages. Wells you excluded are simply not in the table, so you can average
a column or paste it into Prism without first filtering anything out.

The excluded wells are not forgotten — the comment lines at the bottom record
which ones you dropped, along with the gates you used, which markers were on,
and how many cells the file covers:

```
# gates applied (mean intensity)
# GFP,414.065
# RFP,331.107
# markers included,"GFP RFP Cy5"
# wells in this file,93 of 96
# wells excluded,"A1 B2 H12"
# cells in this file,39060
```

Most tools ignore lines starting with `#`. In pandas, use
`pd.read_csv(path, comment="#")`.

Keep that file. If someone asks how you got a number, the answer is in it.

---

## Honest limits

- **One threshold per marker, across every selected well.** The explorer does
  not do per-well or per-plate gating. That's deliberate — comparing wells
  gated differently is usually a mistake — but it does mean that if your plates
  have a strong batch effect you should look at them one at a time (use the
  Plate dropdown).
- **It re-gates; it does not re-segment.** Nuclear outlines and foci counts
  were fixed when the images were processed. If segmentation is wrong, no
  amount of gate-dragging fixes it — go back to the overlays.
- **Excluding wells until the result looks good is not analysis.** Decide what
  counts as a failed well *before* you look at the effect — low cell count,
  visible artefact, failed staining — and apply that rule to every well
  including your controls. Then say in your write-up which wells you dropped
  and why. The exported CSV lists them for you.
- **Dragging gates until p < 0.05 is the same mistake**, more subtly. If your
  conclusion changes when the gate moves a little, the honest finding is that
  the effect is not robust to gating.
- **Cells are not replicates.** 250,000 cells in 96 wells is n = 96 at best,
  and usually fewer. See the statistics section of the main README.

---

## If something goes wrong

| What you see | What to do |
|---|---|
| "No intensity columns found" | This isn't a per-nucleus file. Look for `per_nucleus_measurements.csv`, not a summary. |
| "This file is gzipped and the decompressor didn't load" | You're offline. Ask for a plain `.csv`, or unzip it first. |
| Nothing happens when you drop a file | Drop it on the window, not the button. Or use "Choose a file". |
| A marker won't switch off | At least one has to stay on. |
| The browser stalls on a huge file | Over ~1M nuclei, analyse one plate at a time. |
| Percentages don't match my report | Check the same wells are included and the same markers are on — the header shows both counts. |
