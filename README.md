# FullProf Rietveld Skill (`fullprof-rietveld`)

An expert workflow skill for AI coding assistants (Google Antigravity, Claude Code, Cursor, Codex) and materials scientists performing multi-phase powder X-ray diffraction (XRD) Rietveld refinements with FullProf (`fp2k`).

---

## Overview

This repository packages the **`fullprof-rietveld`** workspace skill, providing end-to-end procedures, robust bug workarounds, single-phase profile decomposition algorithms, and publication-ready matplotlib visualization standards.

```text
fullprof-rietveld/
├── SKILL.md          # Primary agent runbook with YAML frontmatter
├── README.md         # Documentation and usage guide
├── LICENSE           # MIT License
└── .gitignore
```

---

## Key Capabilities

### 1. Staged Refinement Protocol & Physical Guardrails
- **Set 1 Baseline**: Freezes zero-shift, background polynomial, lattice parameters, and profile shapes ($U, V, W, X, Y$) before atomic degrees of freedom are explored.
- **Set 2 Structural**:
  - Damped relaxation factor (`R_at = 0.25–0.50`) to prevent oscillatory least-squares divergence.
  - Coordinate drift bounds ($|\Delta x, \Delta y, \Delta z| \le 0.015$) from reference single-crystal structures.
  - Stoichiometric occupancy clamps ($\text{Occ}_{\mathrm{chem}} \le 1.00$) referencing the invariant transition-metal framework.

### 2. FullProf Bug Fixes & Workarounds
- **`.prf` Column 5 (`Backg`) Downward Shift Bug**:
  When raw intensities in `.dat` exceed the polynomial evaluation at $2\theta_{\min}$ (e.g. from an excluded low-angle sample holder hump), FullProf's `.prf` export routine subtracts $\sim 100\text{ counts}$ from Column 5 across the entire pattern while leaving $Y_{\mathrm{calc}}$ untouched.
  *Fix*: Evaluate the background polynomial $B(2\theta) = \sum_{i=0}^{11} B_i ((2\theta/\text{Bkpos})-1)^i$ directly from `.pcr` coefficients.
- **`.sum` Atomic Parameter Parsing**:
  Avoids splitting by `"---"` which collapses `=> Phase No. X` sections; splits reliably using regex `r"=>\s+Phase No\.\s+\d+"`.
- **Exact Single-Phase Profile Decomposition ($Y_p(2\theta)$)**:
  Zeroes all parameter refinement codewords (`0.00`) and sets `0 !Number of refined parameters` during single-phase calculation runs to eliminate normal matrix inversion singularities (`Singular matrix!!`).

### 3. Publication-Quality Visualization Standards
- Designed for arbitrary sample batches ($N \ge 1$), prioritizing standalone individual plots (`5.5 in` wide):
  1. **Unsubtracted Individual Plots**: $Y_{\mathrm{obs}}$, $Y_{\mathrm{calc}}$, true polynomial $B(2\theta)$, Bragg reflection ticks, and difference curve.
  2. **Background-Subtracted Plots**: Net intensity $Y_{\mathrm{obs}} - B(2\theta)$ and $Y_{\mathrm{calc}} - B(2\theta)$ with zero line, shifted ticks, and shifted difference curves.
  3. **Phase-Decomposed Profile Plots**: Continuous solid, color-coded phase profile curves $Y_p(2\theta)$, each vertically offset above the difference curve.
  4. **Single-Panel Model Comparison Plots**: Overlays both calculated profile fits ($Y_{\mathrm{calc}}^{\mathrm{Set1}}$ and $Y_{\mathrm{calc}}^{\mathrm{Set2}}$, thin solid curves) directly on $Y_{\mathrm{obs}}$, with Bragg ticks and both difference curves on a shared baseline below.
- **Typography & Styling**: Arial font preset, normal font weight, grey difference curves in all cases (`#555555`), per-panel legends in stacked/multi-row figures, borderless translucent top-left text box reporting $\chi^2$ and $R_{\mathrm{wp}}^2$ as pure fractional values (no percentages), bold right-aligned phase labels, and black rectangular legends (`fancybox=False, edgecolor='k'`).

---

## Installation

### In an Antigravity Workspace
Clone or copy this repository into your project's `.agents/skills/` directory:

```bash
mkdir -p .agents/skills
git clone https://github.com/stevenjgomez/fullprof-rietveld-skill.git .agents/skills/fullprof-rietveld
```

Antigravity will automatically discover `fullprof-rietveld` and activate it whenever powder diffraction or FullProf tasks are requested.

---

## License

This project is licensed under the [MIT License](LICENSE).
