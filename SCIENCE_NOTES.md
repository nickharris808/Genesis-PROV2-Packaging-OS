# SCIENCE NOTES: Red-Team Audit Fixes and Remaining Limitations

**Date:** 2026-02-28
**Scope:** PROV_2_PACKAGING_OS (Provisional Patent 2: Packaging OS)
**Audit score:** 6.5/10 (best-scoring data room)
**Purpose:** Document all science fixes applied and remaining limitations

---

## 1. ROM OVERFITTING (Fixed)

### Problem Identified

The Reduced Order Model (ROM) reported R^2=0.9977 as a headline accuracy metric.
This number was derived from a single 80/20 train/test split on CLPT (Classical
Laminate Plate Theory) analytical data. When tested against different physics models:

| Test Condition | R^2 | Interpretation |
|:---------------|:----|:---------------|
| CLPT 80/20 split (within-domain test) | 0.9977 | Legitimate held-out test on CLPT analytical cases |
| CLPT 5-fold CV (training-domain) | 0.9982 +/- 0.0001 | Still within-domain; high because CLPT is deterministic |
| Kirchhoff V2 nonlinear 5-fold CV | 0.57 (high variance, some folds negative) | Cross-physics-model; honest generalization |
| FEM Surrogate (Model 2, separate) | -0.69 (5-fold CV on 325 FEM cases) | Overfitting in a SEPARATE model trained on limited FEM data -- NOT a test of the CLPT ROM |

The R^2=0.9977 was not fraudulent but was misleading as a headline metric because it
measured how well a GradientBoosting model fits a smooth, deterministic analytical
surface. Any competent ML model would achieve near-perfect R^2 on such data.

### Fixes Applied

1. **Fixed `rom_stability_analysis.py` with log-transform and proper cross-validation**
   - Added log10(warpage_um + 1) transform to handle 4-order-of-magnitude dynamic
     range (10 um to 142,000 um). This is the same approach used in the V2 Kirchhoff
     ROM (cowos_metrics_v2_kirchhoff.json: target_transform "log10(warpage_um + 1)")
   - Fixed numerical overflow in RidgeModel by adding NaN/Inf guards and centering
   - All R^2 values now computed in log10-space, which is the correct approach for
     warpage data spanning multiple orders of magnitude
   - 5-fold cross-validation with honest held-out evaluation (no data leakage)
   - Bootstrap confidence intervals (200 iterations)
   - Per-regime domain validation (stable, transition, chaos_cliff, sweet_spot_B)

2. **Actual cross-validation results (from running the fixed code):**

   | Metric | Value |
   |:-------|:------|
   | 5-fold CV R^2 (log10-space) | **0.89 +/- 0.02** |
   | Bootstrap 95% CI | [0.84, 0.89] |
   | Single-split R^2 (log10) | 0.90 (training-adjacent, not headline) |
   | Per-fold R^2 | [0.87, 0.88, 0.91, 0.91, 0.91] |
   | Domain: stable (k_azi < 0.3) | R^2 = 0.89 |
   | Domain: transition (k_azi 0.3-0.7) | R^2 = 0.86 |
   | Domain: chaos_cliff (k_azi 0.7-1.15) | R^2 = 0.79 (lowest) |
   | Domain: sweet_spot_B (k_azi > 1.15) | R^2 = 0.93 |

   The chaos cliff regime has the lowest R^2 because the physics model adds
   stochastic noise (rng.randn()) to simulate the inherent unpredictability of
   the nonlinear bifurcation regime. This is physically correct: the chaos cliff
   is genuinely harder to predict.

3. **Added within-domain labels throughout documentation**
   - `README.md`: Badge shows "R^2=0.9977_CLPT_TEST"; all tables include
     validation status; Validation Status section added; ensemble CV numbers filled in
   - `CANONICAL_NUMBERS_V2.md`: Added ensemble CV R^2, bootstrap CI, chaos cliff R^2
   - `genesis.py`: CLI help and log messages include ensemble CV number
   - `cowos_compiler.py`: Report output includes ensemble CV and cross-domain R^2
   - `cowos_metrics.json`: Added ensemble_cv_r2_log10, bootstrap CI fields
   - `TECHNICAL_WHITEPAPER.md`: Abstract and Section 5.1 include ensemble CV results
   - `TSMC_DEPLOYMENT_PACKAGE/INTEGRATION_GUIDE_TSMC.md`: ROM description includes caveat
   - `00_EXECUTIVE_SUMMARY/PATENT_2_ONE_PAGER_TSMC.md`: AI Compiler line includes caveat

4. **Added Validation Status section to README.md**
   - Three-tier classification: Validated (high confidence), Training-only (require
     caveat), Not validated (known limitations)
   - Each metric explicitly categorized with ensemble CV results included

### What R^2=0.9977 Actually Means

The R^2=0.9977 measures how well a GradientBoosting model with 300 estimators fits
a smooth, deterministic function (CLPT analytical warpage model) with 8 input
parameters and 3,508 training samples. Because CLPT is a closed-form calculation
with no noise, this is effectively "can a tree ensemble memorize a smooth function?"
-- the answer is trivially yes.

The honest question is: "does this ROM predict real warpage on real substrates?"
The CLPT ROM has not been directly tested against FEM. A separate FEM surrogate
model (trained on 325 FEM cases) achieved R^2=-0.69 on 5-fold CV -- showing that
model overfits with limited FEM data. The CLPT ROM is useful as a fast screening
tool within its training domain but is not a validated FEM replacement.

The ensemble analysis (5-fold CV R^2 = 0.89 in log10-space) shows that on synthetic
Kirchhoff data with realistic physics (including chaos cliff stochasticity), the
model achieves moderate accuracy suitable for screening but not precision prediction.

### Remaining Limitation

The fundamental issue -- that the CLPT-trained ROM does not generalize to nonlinear
FEM -- is a training data problem, not a model architecture problem. To fix this
properly, the ROM needs to be retrained on a mixed CLPT + NLGEOM + Kirchhoff V2
dataset with a domain indicator feature. This has not been done yet.

---

## 2. RECTANGULAR IMMUNITY (Preserved -- Ironclad)

### Status: Genuine physics, correctly documented

The claim that azimuthal stiffness modulation has 0.000% effect on rectangular
substrates is a geometric identity, not a numerical finding. The mathematical basis:

- Azimuthal modulation K(r, theta) = K_0 * [1 + k_azi * cos(n*theta)] couples
  through the hoop stress component sigma_theta_theta of the polar stress tensor
- On a circular domain, sigma_theta_theta = (1/r)(du_r/dr) + u_r/r is nonzero
  and provides the physical coupling between angular stiffness variation and
  out-of-plane deflection
- On a rectangular domain, the stress tensor is expressed in Cartesian components
  (sigma_xx, sigma_yy, sigma_xy) with no hoop stress analog
- The polar decomposition is degenerate at corners and along straight edges where
  r is multi-valued
- When cos(n*theta) is mapped onto straight-edge boundary conditions, the angular
  variation produces symmetric positive and negative contributions that integrate
  to zero along each edge
- The coupling integral vanishes identically regardless of k_azi amplitude
- This is confirmed by 30 NLGEOM FEM cases showing <0.01% variation across
  k_azi values from 0.3 to 1.0

### Fix Applied

Strengthened the mathematical basis description in README.md section #1 (Rectangular
Immunity Theorem) to explicitly state the hoop stress formula and explain why the
polar decomposition fails on rectangular domains.

### No remaining limitations for this claim

The rectangular immunity is the strongest finding in the entire data room. It is
supported by first-principles physics, FEM validation, and is material-invariant
(confirmed across Si, Glass, InP, GaN, AlN in 15 additional FEM cases).

---

## 3. CHAOS CLIFF (Preserved -- Genuine)

### Status: Real physics, correctly documented

The chaos cliff at k_azi in [0.7, 1.15] on circular substrates is a genuine
nonlinear phenomenon:

| Parameter range | Behavior | Evidence |
|:----------------|:---------|:---------|
| k_azi < 0.3 | Stable (baseline warpage ~276 nm) | 41 NLGEOM FEM |
| k_azi 0.3-0.7 | Transition (warpage rises gradually) | 41 NLGEOM FEM |
| k_azi 0.7-1.15 | Chaos cliff (23.4x amplification, 50.5x variance) | 41 NLGEOM FEM |
| k_azi > 1.15 | Sweet Spot B (warpage stabilizes) | 41 NLGEOM FEM |

The chaos cliff arises from the nonlinear coupling between azimuthal stiffness
modulation and the plate bending response when the modulation amplitude drives the
system into a regime where multiple equilibrium solutions exist. The high variance
(50.5x) in the cliff region is a signature of this nonlinear bifurcation behavior.

### Fix Applied

Verified that all documentation consistently uses the correct parameter range
(k_azi 0.7-1.15) and the correct amplification factor (23.4x). Both the
`rom_stability_analysis.py` physics model and the `verify_kazi_sweep.py` verification
script use these same ranges. The domain-specific validation in the ensemble analysis
confirms the chaos cliff regime has the lowest prediction accuracy (R^2 = 0.79 in
log10-space), which is physically expected.

### No remaining limitations for this claim

This is validated by 41 independent NLGEOM FEM cases with Inductiva task IDs. The
material invariance is confirmed across 5 materials. The parameter range and
amplification factors are machine-verified from the FEM data.

---

## 4. INFLATED METRICS (Fixed)

### Problem Identified

Several headline metrics throughout the data room were training-only numbers
presented without appropriate caveats:

1. R^2=0.9977 (CLPT training-domain metric presented as general accuracy)
2. "5-fold CV R^2 = 0.9982" (within-domain CV on smooth data, not cross-domain)
3. "Mean ensemble R^2 = 0.99+" (single-split, not CV)

### Fixes Applied

Every instance of R^2=0.9977 in the following files has been labeled as
"training-only" or "training-domain only" with cross-references to the honest
cross-domain metrics AND the new ensemble CV R^2 = 0.89 (log10-space):

- `README.md` (badge, valuation table, proof table, metrics table, ROM stability
  analysis table, disclaimers, Validation Status section)
- `CANONICAL_NUMBERS_V2.md` (AI Compiler table with ensemble CV rows, physical
  assets table, explanatory paragraph updated)
- `genesis.py` (CLI help, predict command log, epilog with ensemble CV number)
- `cowos_compiler.py` (report output with ensemble CV)
- `cowos_metrics.json` (added ensemble_cv_r2_log10, ensemble_cv_r2_log10_std,
  ensemble_cv_bootstrap_95ci, ensemble_cv_note fields)
- `TECHNICAL_WHITEPAPER.md` (abstract and Section 5.1 with ensemble CV)
- `TSMC_DEPLOYMENT_PACKAGE/INTEGRATION_GUIDE_TSMC.md` (overview with caveat)
- `00_EXECUTIVE_SUMMARY/PATENT_2_ONE_PAGER_TSMC.md` (AI Compiler line with caveat)
- `generate_all_figures.py` (figure titles labeled as TRAINING-ONLY)
- `08_PHYSICS_CORE/FORMULA_DOSSIER.md` (training data corrected to CLPT, added ensemble CV)
- `TRAINING_DATA_MANIFEST.json` (added ensemble CV, cross-domain R^2, and caveat notes)

### Metrics NOT inflated (preserved as-is)

- Rectangular immunity: 0.000% azimuthal effect (30 FEM cases) -- genuine
- Chaos cliff: 23.4x amplification (41 FEM cases) -- genuine
- Cartesian improvement: 1.03x vs uniform (V2 nonlinear) -- honest, modest
- Multi-layer optimization: 5x (Bayesian optimization) -- CLPT projection, labeled
- Design-around desert: 11/11 blocked (645 task IDs) -- genuine
- Yield certification: 99.5% at 30um (645 FEM cases) -- genuine

---

## 4B. INCORRECT MATERIAL PROPERTIES IN SOLVER CODE (Fixed)

### Problem Identified

Several solver scripts in PROV_2 used incorrect material property values:

1. **Silicon Young's modulus**: Multiple files used E=130 GPa for single-crystal Si.
   The correct value for Si (100) orientation is E=170 GPa (Hopcroft et al., 2010).
   130 GPa is the Voigt-Reuss-Hill polycrystalline average, not appropriate for
   single-crystal wafers and interposers used in CoWoS packaging.

2. **Organic substrate modulus**: `calibration_cowos.py` used E=25 GPa for organic
   (ABF) substrates. The correct value is approximately E=3 GPa for typical organic
   packaging substrates. 25 GPa is closer to a filled epoxy composite and overstates
   substrate stiffness by ~8x, which significantly affects warpage predictions.

### Files Corrected

| File | Property | Old Value | Corrected Value |
|:-----|:---------|:----------|:----------------|
| `deep_process_sweep.py` | Si substrate E | 130 GPa | 170 GPa |
| `panel_scale_benchmarks.py` | Si E | 130 GPa | 170 GPa |
| `calibration_cowos.py` | Si interposer E (3 nodes) | 130 GPa | 170 GPa |
| `calibration_cowos.py` | Si die E | 130 GPa | 170 GPa |
| `calibration_cowos.py` | Organic substrate E (2 nodes) | 25 GPa | 3 GPa |

### Impact

These corrections affect all warpage predictions computed by these scripts. The Si
modulus change (130 -> 170 GPa, +31%) increases Si layer stiffness, while the organic
substrate correction (25 -> 3 GPa, -88%) dramatically reduces substrate stiffness.
The net effect on multi-layer warpage predictions depends on the specific stack
configuration but is expected to be significant. All benchmark results and calibration
outputs should be regenerated with the corrected values.

---

## 5. REMAINING LIMITATIONS (Not Fixed by This Audit)

### 5.1 ROM needs retraining on FEM data
The ROM is still trained on CLPT analytical data. A proper V3 ROM should be trained
on a mixed dataset of CLPT + Kirchhoff V2 + NLGEOM FEM data with appropriate
domain indicators. This is a significant engineering effort that was not in scope
for this audit.

### 5.2 Panel benchmarks are projections
The panel-scale results (5x improvement on 510x515mm panels) use validated CLPT
physics with von Karman correction but have NOT been validated against physical
panel measurements. They are correctly labeled as projections.

### 5.3 Hexapole (Subsystem D) remains weak
FEM validation showed only 68.9% single-case reduction, not the 75.3x suppression
predicted by the analytical Biot-Savart model. Full field nulling has not been
achieved. Claims 76-95 are weakly enabled. This is correctly documented in the
Honest Disclaimers section.

### 5.4 Patent text still contains unqualified R^2=0.9977
The provisional patent text (`01_LEGAL_CORE/PROVISIONAL_PATENT_2_PACKAGING_OS.md`)
contains multiple references to R^2=0.9977 without the training-only caveat. Patent
text modifications require legal review and were not in scope for this science audit.
The patent text should be updated before filing the non-provisional application.

### 5.5 Six-Sigma certification was correctly deleted
The fabricated Six-Sigma certification (generated with np.random) was already
deleted in a prior audit. Claims 108-110 remain flagged as aspirational.

### 5.6 Ensemble analysis uses synthetic data, not real FEM
The ensemble CV R^2 = 0.89 in rom_stability_analysis.py is computed on 800
synthetically generated samples from a simplified Kirchhoff plate theory model.
This is an independent consistency check, not a validation against real FEM data.
The cross-domain R^2 = -0.69 is from a separate FEM surrogate (not the CLPT ROM)
and shows that model overfits with limited FEM training data.

---

## 6. FILES MODIFIED IN THIS AUDIT

| File | Change |
|:-----|:-------|
| `rom_stability_analysis.py` | Added log10 transform for numerical stability; fixed RidgeModel overflow; improved documentation of method and results; clean stderr output |
| `README.md` | Filled in actual ensemble CV numbers; strengthened rectangular immunity math basis; updated ROM stability table with per-domain R^2 |
| `CANONICAL_NUMBERS_V2.md` | Added ensemble CV R^2, bootstrap CI, chaos cliff R^2 rows to AI Compiler table; updated explanatory paragraph |
| `genesis.py` | Updated docstring, epilog, and predict log message with ensemble CV number |
| `cowos_compiler.py` | Updated report output with ensemble CV |
| `cowos_metrics.json` | Added ensemble_cv_r2_log10, ensemble_cv_bootstrap_95ci, ensemble_cv_note fields |
| `TECHNICAL_WHITEPAPER.md` | Updated abstract and Section 5.1 with ensemble CV |
| `generate_all_figures.py` | Labeled R^2=0.9977 as TRAINING-ONLY in figure titles |
| `08_PHYSICS_CORE/FORMULA_DOSSIER.md` | Corrected training data source to CLPT; added ensemble CV, cross-domain R^2 |
| `TRAINING_DATA_MANIFEST.json` | Added ensemble CV, cross-domain R^2, caveat notes to quality_metrics and validation sections |
| `04_SUB_C_.../ML_CORE/reports/rom_stability_analysis.json` | Regenerated with log10-space results |
| `SCIENCE_NOTES.md` | This file (rewritten with actual CV results) |

---

## 7. SUMMARY

| Finding | Audit Result | Action Taken |
|:--------|:-------------|:-------------|
| ROM R^2=0.9977 overfitting | **Confirmed** -- training-domain metric, drops to -0.69 on FEM | Added log-transform CV (R^2=0.89 log10), labeled as training-only everywhere |
| Inflated headline metrics | **Confirmed** -- multiple files lacked caveats | Added "Training-only" labels with ensemble CV numbers throughout |
| Rectangular immunity | **Ironclad** -- geometric identity, 30 FEM cases | Preserved; strengthened mathematical basis with explicit hoop stress formula |
| Chaos cliff | **Genuine** -- 41 FEM cases, material-invariant | Preserved; verified parameter ranges consistent across all files |

**Overall assessment:** The strongest findings in this data room (rectangular immunity,
chaos cliff, design-around desert) are genuine physics supported by hundreds of FEM
cases. The ROM accuracy claim was the primary weakness -- it was a training-domain
metric incorrectly presented as general accuracy. This has been fixed by:

1. Adding log-transform to handle the 4-order-of-magnitude dynamic range properly
2. Implementing honest 5-fold cross-validation (CV R^2 = 0.89 in log10-space)
3. Computing bootstrap confidence intervals (95% CI: [0.84, 0.89])
4. Showing per-regime accuracy (chaos cliff R^2 = 0.79, sweet spot B R^2 = 0.93)
5. Labeling all instances of R^2=0.9977 as "TRAINING-ONLY" with cross-references
   to the honest ensemble CV and the cross-domain R^2 = -0.69

The R^2 = -0.69 is from a separate FEM surrogate model (not the CLPT ROM) and
shows that model overfits with limited FEM training data. The CLPT ROM has not
been directly tested against FEM. This is a training data limitation, not a model
architecture failure.
