---
name: fullprof-rietveld
description: >-
  Executes, parses, troubleshoots, and plots multi-phase powder XRD Rietveld refinements
  using FullProf (fp2k). Use when setting up .pcr/.dat files, refining lattice/profile/atomic
  parameters, decomposing individual phase profiles, or generating publication-ready Rietveld figures.
---

# FullProf Rietveld Refinement & Visualization Workflow

## 1. Environment & Binary Execution
- **FullProf Executable**: `/Users/stevengomezalvarado/FullProf_macOS_February2025/fp2k`
- **File Naming Constraint**: `fp2k` requires the intensity file (`.dat`) and control file (`.pcr`) in the working directory to share the exact same basename (`{tag}.pcr` and `{tag}.dat`), and `{tag}` must not contain dots (`.`).

---

## 2. Staged Refinement Protocol & Physical Guardrails
Refinements should proceed in discrete, reproducible stages:
1. **Set 1 Baseline (`set1_baseline/`)**: Freeze instrument zero-shift, background polynomial, lattice parameters, and profile parameters ($U, V, W, X, Y$) before refining atomic parameters.
2. **Set 2 Structural Refinements (`set2_structural/`)**:
   - **Relaxation Damping**: Set `R_at = 0.25` to `0.50` in the `.pcr` control line to prevent oscillatory divergence.
   - **Coordinate Drift Guard**: Enforce $|\Delta x, \Delta y, \Delta z| \le 0.015$ from crystallographic reference coordinates.
   - **Stoichiometric Occupancy Clamp**: Fix the heavy transition-metal framework occupancy at stoichiometric $1.00$ as the scale reference. Clamp refined alkali/pnicogen site occupancies to $\text{Occ}_{\mathrm{chem}} \le 1.00$, where FullProf's normalized occupancy is:
     $$\text{Occ}_{\mathrm{norm}} = \text{Occ}_{\mathrm{chem}} \times \frac{m_{\mathrm{site}}}{m_{\mathrm{general}}}$$
   - **Automatic Model Comparison**: Whenever Set 2 structural refinements are run, automatically compare Set 1 baseline vs. Set 2 candidate structural models (`M0`, `M1`, `M2`, `M3`, etc.) and generate `set1_vs_set2_model_comparison.png` to evaluate improvements in $\chi^2$, $R_{\mathrm{wp}}^2$, and refined structural parameters across all analyzed samples.

---

## 3. Parsing & Data Extraction Pitfalls

### 3.1 Critical `.prf` Background Column Bug
- **Never use Column 5 (`Backg`) of `.prf` verbatim** for plotting background curves or computing background-subtracted intensities. When low-angle intensities in `.dat` exceed the polynomial evaluation at $2\theta_{\min}$, FullProf's `.prf` exporter shifts Column 5 downward by $\approx 100\text{ counts}$ while keeping $Y_{\mathrm{calc}}$ intact.
- **Mandatory Procedure**: Parse `Bkpos` and the 12 polynomial coefficients ($B_0 \dots B_{11}$) directly from the `.pcr` file and evaluate the true polynomial baseline:
  $$B(2\theta) = \sum_{i=0}^{11} B_i \left(\frac{2\theta}{\text{Bkpos}} - 1\right)^i$$

### 3.2 Parsing `.sum` Atomic Tables
- FullProf surrounds `=> Phase No. X` headers with dashed separator lines (`---`). Do **not** split `.sum` by `"---"`; split using the regex `r"=>\s+Phase No\.\s+\d+"` so each phase header remains paired with its atomic parameter block.

### 3.3 Decomposing Individual Phase Profiles ($Y_{\mathrm{calc}, p}$)
To extract exact single-phase calculated profiles from FullProf without triggering `Singular Matrix` errors:
1. Create a temporary directory and copy `{sid}.dat` to `{tag}.dat`.
2. In `{tag}.pcr`, set `NCY = 0`, set `0 !Number of refined parameters`, and **replace all parameter codeword lines with `0.00`** (background, zero, scale, $U, V, W, X, Y$, cell, `Pref1`, and atom codewords).
3. Run `fp2k` with all phase scales set to `0.0000000E+00` to obtain the internal baseline $Y_{\mathrm{none}}(2\theta)$.
4. For each phase $p \in \{1 \dots N_{\mathrm{phases}}\}$, run `fp2k` with only phase $p$ keeping its refined scale factor (all other phase scales `0.0000000E+00`). Parse tab-delimited data lines (`line.split('\t')` where `parts[1].strip()` is non-empty) and compute:
   $$Y_{p}(2\theta) = \max\left(0,\, Y_{\mathrm{calc}}^{(p)}(2\theta) - Y_{\mathrm{none}}(2\theta)\right)$$
5. Export the decomposed profiles to `phase_profiles_{sid}.csv` for reproducibility.

---

## 4. Publication Figure Standards for Rietveld Refinements

Refinement pipelines may process a single sample ($N = 1$) or an arbitrary batch ($N$ samples). Always generate standalone individual plots for each sample `sid` as the primary deliverable; stacked multi-row plots are optional and should not be assumed as the default layout:

1. **Automated Figure Suite (Generated per Sample `sid`)**:
   - **Unsubtracted Individual Plot** (`rietveld_refinement_{set}_{sid}.png`, `figsize=(5.5, 4.2)`): $Y_{\mathrm{obs}}$, $Y_{\mathrm{calc}}$, dashed grey $B(2\theta)$, Bragg tick rows (`tick_base = -0.05 * max_y`, `tick_spacing = 0.065 * max_y`), and difference curve (`diff_offset = -0.28 * max_y`, `ylim = [-0.38 * max_y, 1.15 * max_y]`).
   - **Background-Subtracted Individual Plot** (`rietveld_refinement_{set}_{sid}_bkg_subtracted.png`, `figsize=(5.5, 4.2)`): Net intensity $Y_{\mathrm{obs}} - B(2\theta)$ and $Y_{\mathrm{calc}} - B(2\theta)$ with a dotted zero line at $y=0$, shifted Bragg ticks (`tick_base = -0.16 * max_y`, `tick_spacing = 0.08 * max_y`), and shifted difference curve (`diff_offset = -0.48 * max_y`, `ylim = [-0.68 * max_y, 1.15 * max_y]`).
   - **Phase-Decomposed Individual Plot** (`rietveld_refinement_{set}_{sid}_phase_profiles.png`, height scaled dynamically with $N_{\mathrm{phases}}$, e.g. `figsize=(5.5, 3.6 + 0.8 * N_phases)`): Replaces Bragg ticks with continuous **solid** (`linestyle='-'`), color-coded phase profile curves $Y_p(2\theta)$, each vertically offset (`phase_start = -0.35 * max_y`, `phase_spacing = 0.45 * max_y`) above the difference curve.
   - **Set 1 vs. Set 2 Model Comparison Plot** (`set1_vs_set2_comparison_{sid}.png`, `figsize=(5.5, 4.4)`): Standalone single-panel comparison for each sample displaying experimental data $Y_{\mathrm{obs}}$ (black open circles, `markersize=1.6`, `markeredgewidth=0.45`), both calculated profile fits ($Y_{\mathrm{calc}}^{\mathrm{Set1}}$ solid red `#E41A1C`, `linewidth=0.7` and $Y_{\mathrm{calc}}^{\mathrm{Set2}}$ solid blue `#1E88E5`, `linewidth=0.7`) overlaid directly atop the observed data to reveal subtle peak intensity divergences, Bragg reflection ticks, and both difference curves ($\Delta Y^{\mathrm{Set1}}$ and $\Delta Y^{\mathrm{Set2}}$, `linewidth=0.55`) plotted atop a shared difference baseline (`diff_offset`) below the ticks. An optional multi-row stacked overview (`set1_vs_set2_model_comparison.png`) can also be generated.

2. **Annotations, Labels, & Legends**:
   - **Top-Left Info Text**: Print **only two lines**:
     - Line 1: Sample label (e.g., `YK729 (Rh-rich as-grown)`)
     - Line 2: Goodness of fit reporting **$\chi^2$ and $R_{\mathrm{wp}}^2$** as a pure fractional value without percentages (e.g., `$\chi^2 = 1.15$, $R_{\mathrm{wp}}^2 = 0.057$`), omitting phase weight fractions and $R_B$.
     - Format with a **borderless translucent white background**: `bbox=dict(boxstyle='square,pad=0.25', facecolor='white', edgecolor='none', alpha=0.75)`.
   - **Phase Tick / Curve Labels**: Place on the right (`x = 89.2°`, `ha='right'`) in **bold** math/text (`\mathbf{KRh_2P_2}`, `fontweight='bold'`) with a white background box (`facecolor='white'`, `edgecolor='none'`, `zorder=10`) so ticks or lines never obscure the text.
   - **Typography & Legend**: Strictly follow Global Rule 6 (Arial font preset with `UnicodeFonts.get_xheight` iceberg patch, normal font weight on axes/titles, and sharp black rectangular legend `fancybox=False, edgecolor='k'`).
