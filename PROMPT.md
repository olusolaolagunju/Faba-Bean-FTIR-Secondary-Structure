# Prompt for FTIR secondary-structure analysis

**How to use:** paste everything below the line into Claude (or another AI assistant). Attach `FTIR-ALKALINE.ipynb` (the reference method) and your own data files (CSV or Excel). Fill in the parts in [square brackets] first.

---

I am attaching a reference Jupyter notebook (`FTIR-ALKALINE.ipynb`) and my own FTIR data files. The notebook analyzes the protein secondary structure of a faba bean protein extract from the amide I band. Please write **one Jupyter notebook** that applies the **same method** to my files. Run every step internally and save all the figures and tables listed at the end.

## My data
- Samples / treatments: [e.g., Control, Treatment A, Treatment B]
- Replicate files per sample: [e.g., "SAMPLE 1.csv" to "SAMPLE 6.csv"]
- Column 1 = wavenumber (cm⁻¹). Column 2 = [absorbance / % transmittance]. Lines starting with `#` are comments.
- If the data are % transmittance, convert to absorbance with A = 2 − log10(%T) before anything else.
- Use relative file paths (e.g., `data/SAMPLE 1.csv`), never paths from my computer.

## Method to reproduce (from the reference notebook, final cells)
1. **Average the replicates** of each sample point by point. Check that all replicates share the same wavenumber axis; interpolate if they do not. Save `<SAMPLE>_average.csv`.
2. **Full spectrum figure:** overlay the average absorbance spectra of all samples, 4000–400 cm⁻¹, x-axis reversed (high to low wavenumber).
3. **Amide I window:** keep 1580–1710 cm⁻¹ (1580–1720 cm⁻¹ for peak detection).
4. **Baseline:** straight line between the two end points of the window, subtracted. If any value is then negative, shift the curve up so the minimum is zero.
5. **Second derivative and peak detection:** Savitzky–Golay filter, window length 13, polynomial order 3, `deriv=2`. Invert the sign, then find peaks with `scipy.signal.find_peaks` (prominence 1e-6, minimum distance 3 points). Report the detected peak positions.
6. **Assign each detected peak** to a structure using the ranges in the reference notebook (after Kong & Yu, 2007):
   - 1620–1640 cm⁻¹: β-sheet (about 1620 is intermolecular β-sheet)
   - 1640–1650 cm⁻¹: random coil
   - 1650–1660 cm⁻¹: α-helix
   - 1660–1670 cm⁻¹: disordered α-helix / 3₁₀-helix (counted with α-helix)
   - 1670–1685 cm⁻¹: β-turn
   - 1685–1700 cm⁻¹: antiparallel β-sheet (counted with β-sheet)
   - Below 1620 cm⁻¹ (e.g., 1601 and 1610): side chains, excluded from the fit
7. **Curve fitting:** one Lorentzian band per assigned peak, `amp · wid² / ((x − cen)² + wid²)`, plus a linear term `b0 + b1·x`. Fit with `scipy.optimize.curve_fit` and bounds:
   - Centre: detected position ± 1 cm⁻¹ (starting value: the detected position).
   - Half-width: 3 to 12.5 cm⁻¹ (start 6).
   - Amplitude: 0 to 2 × the maximum corrected absorbance. Each α-helix band must be at least 2% of that maximum.
   - `maxfev = 500000`.
8. **Areas:** integrate each fitted band numerically (`scipy.integrate.simpson`). Sum the bands by structure. Percentage = structure area / total area of all bands × 100.

## The bound rule: do this automatically
I tuned the bounds by trial and error until every fitted value fell **inside** its allowed range, never on a limit. For example, if a band's centre is allowed between 1620 and 1622, the fitted centre must not be 1620.00 or 1622.00. A value sitting on a limit means the fit wanted to go further, so the bound, not the data, decided the result.

The notebook must enforce this in a loop:
- After each fit, check every fitted centre, half-width and amplitude. Flag any value within 0.05 cm⁻¹ (centres and widths) or 0.1% of the limit (amplitudes) of a lower or upper bound.
- If a **centre** is on a bound: widen that band's centre range by 1 cm⁻¹ on that side only. The centre must also stay inside the structure range from step 6, strictly within it, not on its edge. Then refit.
- If a **width** is on a bound: widen that bound by 1 cm⁻¹ (half-width never below 2 or above 20 cm⁻¹), or consider whether the band should be split or merged. Then refit.
- Repeat until nothing touches a limit, or for at most 20 rounds.
- Print a table of each band's final bounds, fitted value and distance from each bound. If the rule still fails, say so clearly. Do not hide it.
- Keep the structure ranges in one editable dictionary at the top of the notebook so I can adjust them.

## Statistics (run internally)
1. **Fit quality** for each sample: RMSE, R², AIC = n·ln(RSS/n) + 2k, BIC = n·ln(RSS/n) + k·ln(n).
2. **Fit stability:** refit 10 to 50 times with randomly perturbed starting values (amplitude and width ±3%, centre ±1 cm⁻¹, fixed random seed). Report the mean ± SD of each structure %.
3. **Replicate variation:** run the whole pipeline on each replicate separately as well as on the average. Report the mean ± SD (n = number of replicates) for each structure %.
4. **Comparison between samples** (if there are 2 or more): one-way ANOVA for each structure, then a Tukey HSD test (α = 0.05). Show significance letters on the bar chart. Check the assumptions with a Shapiro–Wilk test on the residuals and Levene's test. If they fail, use Kruskal–Wallis with Dunn's test instead.

## Outputs to save (figures as PNG at 600 dpi, x-axis reversed)
- `full_spectra_overlay.png`: average absorbance spectra of all samples.
- `<SAMPLE>_peak_detection.png`: top, corrected amide I with detected peaks marked; bottom, inverted second derivative with peaks labelled.
- `<SAMPLE>_secondary_structure.png`: raw spectrum (black), corrected spectrum (blue), total fit (red dashed), and filled bands coloured by structure (β-sheet #1f77b4, α-helix #d62728, random coil #2ca02c, β-turn #ff7f0e). Label each band with its centre. The legend shows each structure's %, and the title is the sample name.
- `<SAMPLE>_residuals.png`: corrected minus fit.
- `combined_panels.png`: one figure with panel (a) as the spectra overlay and panels (b), (c)… as each sample's secondary-structure plot, laid out in a grid with bold letters a, b, c… at the lower left of each panel.
- `structure_barchart.png`: % of each structure per sample, mean ± SD error bars, with Tukey letters.
- `results_summary.csv`: per sample, the % of each structure (average-spectrum fit and replicate mean ± SD), RMSE, R², AIC, BIC, and the perturbation SD.
- `fitted_bands.csv`: per sample and band, the structure, centre, half-width, amplitude, area, bounds, and a "touches bound" flag.

## Reference values for checking
Run on the alkaline data (`absorbance/ALKALINE_average.csv`), the reference notebook gives: β-sheet 38.32%, α-helix 15.24%, random coil 30.01%, β-turn 16.42%, RMSE 0.00090, AIC −1779.20, BIC −1695.82. Reproduce these values first to confirm the method is implemented correctly. If the automatic bound check then adjusts any bound, the numbers may change slightly; report both.

Explain each step in a short markdown cell so I can follow it. Put all adjustable settings (file names, windows, Savitzky–Golay settings, prominence, bounds, structure ranges) in one settings cell at the top.
