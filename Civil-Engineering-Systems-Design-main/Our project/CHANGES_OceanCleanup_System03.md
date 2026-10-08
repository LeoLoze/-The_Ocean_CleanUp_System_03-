# Changes to `OceanCleanup_System03.ipynb`

This file documents the changes made to the existing code of `OceanCleanup_System03.ipynb`, since the first run of the optimization.
So we can trace what was changed, why, and what is still open. Section names refer to the headings in the notebook.

---

## 1. Constants (sections *Design Variables and Bounds*, *Constraints*, *Objective Functions*)

**What changed:** only the comments. All values were already the same as in `Constants_and_Sources.ipynb`.

- Every constant now names its source in the comment (e.g. `# ... - Filho (2021)`), matching `Constants_and_Sources.ipynb`.
- The header "ALL VALUES ARE PLACEHOLDERS" was replaced by "Values and sources as documented in Constants_and_Sources.ipynb".
- The comment for `barrier_length` now cites Verhagen (2024).

---

## 2. Preference curves and preference functions (section *Preference Curves and Preference Functions*)

### Problems in the previous version
- The control points were copied from the artificial reef example (e.g. cost 150,000–4,250,000 while our cost is 7.6–124 M€/yr). Almost every design fell outside the curves, so `pchip_interpolate` extrapolated to scores far outside 0–100.
- The fishing curve had the reef's hump shape (sediment trapping); fragmentation increased instead of decreased.
- `pref_func_2` to `pref_func_5` stored the objective in one variable name and used another (`ride`/`PR`, `sed`/`fish`, `safe`/`eco`, `econ`/`FPF`), which raised a `NameError`.

### New version
Each curve is **linear with two points**: 100 = the value the stakeholder considers ideal, 0 = the value the stakeholder considers unacceptable. Values beyond these limits are clipped to 100 or 0.

| Objective | 100 (ideal) | 0 (unacceptable) | Basis |
|---|---|---|---|
| Cost [M€/yr] | 7.5 | 85 | 7.5 = charter cost of one system (`charter_cost`); 85 ≈ 10-system fleet (upper bound of `x3`) |
| Plastic removal [t/yr] | 7200 | 0 | 90 % of ~80,000 t (`plastic_density` × 1.6 million km²) within 10 years |
| Fishing interference [km²] | 0 | 40 | 10-system fleet at today's settings closes ~42 km² (judgement) |
| Ecological risk [animals/yr] | 1e6 | 1e8 | ≈ lowest attainable value; ≈ double today's single system (judgement) |
| Fragmentation [t/yr] | 0 | 3 | 10-system fleet at today's settings produces ~3 t/yr (judgement) |

Code changes in the cell:
- `prefs_before` and `pref_after` contain the points above (`pref_after` is a copy, to be updated after the stakeholder game).
- New helper `pref_score(pref, value)`: `pchip_interpolate` clipped to [0, 100]. All five `pref_func_*` use it; the variable name mismatches are fixed.
- The plot shows each curve only between its two points, in a square plot area, so every curve is a 45° slope. Today's design (`X_irl`) is marked with a red cross.
- The cell prints the preference scores of today's design.
- The markdown above the cell now explains the curves for System 03 instead of the reef.

### Update: three points per curve
The linear curves assumed that every unit of improvement is worth the same to a stakeholder. Each curve now has a third point at a score of **50** (the value at which the stakeholder is halfway satisfied); its objective value sets the curve shape. The 0 and 100 limits are unchanged:

| Objective | Points (objective → score) | Shape |
|---|---|---|
| Cost [M€/yr] | 7.5 / 55 / 85 → 100 / 50 / 0 | slow drop, then steep (budget) |
| Plastic removal [t/yr] | 0 / 1100 / 7200 → 0 / 50 / 100 | diminishing returns |
| Fishing interference [km²] | 0 / 25 / 40 → 100 / 50 / 0 | flat, then steep |
| Ecological risk [animals/yr] | 1e6 / 1.5e7 / 1e8 → 100 / 50 / 0 | log-like (order of magnitude) |
| Fragmentation [t/yr] | 0 / 0.5 / 3 → 100 / 50 / 0 | steep first |

- The score of the middle point is fixed at 50 for all stakeholders so every middle point has the same meaning and can be elicited with one question in the stakeholder game ("at which value would you be halfway satisfied?").
- The 50-points are judgement and should be revisited in the stakeholder game (`pref_after` is still a copy of `prefs_before`).
- `pref_score` now clamps the **objective value** to the range of the control points before interpolating, instead of clipping the score afterwards. With 3+ points, `pchip_interpolate` extrapolates with a cubic that bends back (e.g. a bycatch of 2e8 scored 46 instead of 0), so clipping the score was no longer sufficient.
- The "45° slope" comments in the preference and optimisation plots were removed; the markdown table was rewritten without the outdated "10-system fleet" numbers.

---

## 3. Optimisation (new section *Optimisation*)

New markdown cell (how the `GeneticAlgorithm` is used) and new code cell, following the reef example as closely as possible. Differences from the reef code:

- `'var_type_mixed': var_type_mixed` instead of `'var_type': 'real'`, because `x3` (number of systems) is an integer.
- The loop variable `pref_score` was renamed to `pref_value`, so it does not overwrite the `pref_score` function used by the preference functions.
- Runs `'minmax'` and `'a-fine'`. `'tetra'` is only mentioned in a comment because it needs the online Tetra solver and credentials. A third marker and colour are already defined for it.
- The plots keep the square, 45° layout of the preference curves.

---

## 4. Constraints (section *Constraints*)

### Problem in the previous version
No design within the bounds satisfied all three constraints (0 % feasible in a sample of 20,000 designs), so every design in every GA generation was infeasible and the results were meaningless. The GA treats a return value **> 0 as a violation**.

### Constraint 1 – Maximum cable tension

| | Previous | New |
|---|---|---|
| Formula | `Cd * (L * x1) * x4**2 * (x2 / L) * (1 / x5)` | `tension = 0.5 * rho_water * Cd * (x1 * x2) * x4**2 / 2 / 1000` |
| Returned | the force itself (always > 0, so always violated) | `tension - max_tension` |

- Uses the drag formula with the same frontal area (skirt depth × mouth width) as the cost objective; the drag is shared by the two towing cables and converted to kN.
- The `1 / x5` (mesh size) factor was removed: `Cd = 1.2` already comes from a calibration of netted screens (Gonzalez Jimenez et al., 2023).

### Constraint 2 – Maximum waste bag volume

| | Previous | New |
|---|---|---|
| Formula | `K_vol * x1 * x2 * sqrt(L**2 * x2**2)` (water volume inside the U, 6.7e7 – 1.3e10 m³) | plastic collected by one system between two extractions: `removal_per_system * extraction_interval / 365 / plastic_bulk_density` |
| Returned | `max_volume - volume` (sign reversed, never violated) | `bag_volume - max_volume` |

- `K_vol` is no longer used.
- New constants (**placeholders, need a source**): `extraction_interval = 7` days, `plastic_bulk_density = 150` kg/m³.
- Uses `objective_function_2`, which is defined later in the notebook; this works because constraints are only evaluated when the GA runs.
- With the current values this constraint is **never active** (the bag fills to at most ~83 of 1540 m³ in 7 days).

### Constraint 3 – Vessel fuel autonomy

| | Previous | New |
|---|---|---|
| Formula | `Fuel_max - (K_base * x3 * x4**3 + K_weight * 0.50 * x4**2 * x3)` | `fuel_per_hour * trip_duration - Fuel_max` with `fuel_per_hour = K_base * x4**3 + K_weight * 0.50 * x4**2` |
| Returned | sign reversed (always violated); compared litres per hour with total litres | fuel over one campaign minus the fuel capacity |

- `x3` was removed from the fuel use: each system has its own vessels and tanks.
- New constant (**placeholder, needs a source**): `trip_duration = 60 * 24` h (60-day campaign).
- The factor `0.50` in the `K_weight` term was kept as it was; its meaning is not documented.

### Result
- 81.7 % of random designs are now feasible; today's design (`X_irl`) satisfies all three constraints.
- Constraint 1 is active for deep skirts combined with a wide mouth and high speed (16.5 % of designs), constraint 3 for towing speeds above ~0.73 m/s (3.6 %).
- The GA runs normally with both aggregation methods.

---

## 5. Objective 4 – Ecological risk recalibrated

### Problem in the previous version
`animal_density` (2.1e-3 per m³) was derived from the neuston count (1/180 of the plastic pieces per km²) and applied to the whole swept water volume with a 63 % capture probability. One System 03 came out at ~9e7 animals/yr, about 4,000 times the observed bycatch. Neuston is not bycatch (no systematic impact found by the EIA and Egger et al., 2025).

### New version
- Bycatch = sea surface area swept × `bycatch_ref` × depth, speed and mesh factors that equal 1 for System 002 (3 m, 0.75 m/s, 10 mm).
- `bycatch_ref` = 1.0 per km² swept, calibrated on ~13,800 individuals of primary bycatch over ~12,900 km² swept by System 002 in campaigns 1–12 (EIA Table 5-7, §5.2.5).
- New constants `bycatch_ref`, `depth_ref`, `speed_ref`; `animal_density` removed. Derivation, plausibility checks and limitations are in `Constants_and_Sources.ipynb` (new section *Calibration of `bycatch_ref`*, references 18–20 added).
- Range is now ~920 to ~530,000 animals/yr; today's design gives ~20,000 (per-tonne cross-check: ~30,000).
- Preference points changed to 2,000 / 25,000 / 250,000 → 100 / 50 / 0, so today's system scores ~56 instead of ~1.

---

## 6. Cost preference curve lowered (section *Preference Curves and Preference Functions*)

### Problem in the previous version
One system costs only 7.7–12.4 M€/yr (7.5 M€ of it is the fixed charter), so the cost is mainly set by the number of systems. With the points 7.5 / 55 / 85 a fleet of 4–5 systems still scored about 70, so cost never restricted the minmax design and most of the plotted curve covered fleets of 5–10 systems.

### New version
- Cost points changed to **7.5 / 30 / 60 → 100 / 50 / 0** in `prefs_before` and `pref_after`: halfway satisfied at about three systems at today's settings, unacceptable at about six.
- The curve is now nearly linear (the 50-point lies slightly below the linear midpoint of 34). The markdown table was updated.
- The budget limits are judgement and still need a source.
- Effect: the minmax optimum moves from 4–5 systems to 3 systems (about 25 M€/yr), with almost the same plastic removal. Today's design scores 93 instead of 98 on cost.

---

## 7. Optimisation: several runs and corrected a-fine result (section *Optimisation*)

### Problems in the previous version
- A single GA run per method. Runs with other random seeds gave different designs (e.g. 3 or 4 systems for minmax).
- The a-fine result was not an optimised design. a-fine scores are relative (the best design of every generation scores -100), so the GA never replaces the best design of its first, random generation and always stops after exactly `max_stall` generations with the "too fast convergence" warning. A larger `max_stall` alone does not change the returned design.

### New version
- Each method is run `n_runs = 5` times with a fixed seed per run (`np.random.seed`), so the results are reproducible. One line is printed per run instead of the generation log.
- `max_stall` is set per method: 30 for minmax, 60 for a-fine (for a-fine this is the number of generations).
- minmax: the run with the lowest score is kept.
- a-fine: a wrapper around `objective` stores the populations the GA evaluates; the best feasible design of the **final** generation is taken (with `a_fine_aggregator`), and the winners of the five runs are compared with one more a-fine aggregation. The warning is hidden for a-fine only, because it always appears.
- The GA code in `genetic_algorithm_pfm` was not modified.
- New imports: `contextlib`, `io`, `a_fine_aggregator`. New variable `optimal_designs` holds the best design per method.
- Result: four of five minmax runs agree (3 systems, 0.37 m/s; one run ends in a local optimum with 4 systems). All five a-fine runs agree (1 system, 800 m spacing, 0.30 m/s, 10 mm mesh).

---

## 8. Objective 5 – net fragmentation (sections *Objective Functions*, *Preference Curves and Preference Functions*)

### Problem in the previous version
Objective 5 only counted the microplastic that the system itself creates (plastic met × share smaller than the mesh × speed factor). Plastic removal did not appear in it, so the best design for citizens was the one that meets the least plastic. The a-fine optimum was therefore one small, slow system that removed only ~137 t/yr, and not deploying at all would have scored 100 for citizens. In reality, plastic that is left floating keeps breaking down into microplastic.

### New version
- `objective_function_5` now returns the **net** microplastic formation: created by the systems (unchanged formula) minus avoided by removal.
- Avoided = plastic removed (`objective_function_2`) × `natural_frag_rate`.
- New constant `natural_frag_rate = 0.03` per year: share of the floating plastic mass that degrades into microplastic per year, the best-fit value of Lebreton, Egger & Slat (2019). Added to `Constants_and_Sources.ipynb` (reference 21).
- Only one year of avoided fragmentation is counted per tonne removed, although that plastic would have kept fragmenting in later years. The benefit is therefore a conservative estimate.
- A negative value means more microplastic is avoided than created. The objective is still minimised. Its label is now "Net Fragmentation"; it depends on all five variables (`x1` enters through removal).
- Range is now -191 to -1.7 t/yr; today's design gives -12.0 t/yr (1.7 created, 13.7 avoided). The systems create fragments equal to less than 1 % of the mass they remove, so the value is negative for every design.
- Preference points changed from 0 / 0.5 / 3 to **-216 / -33 / 0 → 100 / 50 / 0** in `prefs_before` and `pref_after`: 0 = no net benefit, 100 = cleanup target of 7200 t/yr removed (× 3 %), 50 = the first ~1100 t/yr removed. These mirror the plastic-removal curve and are judgement.
- The markdown description of objective 5, the objective table and the preference table were rewritten. The variables of the citizens' objective were updated in `Stakeholder_Objectives_and_optimisation/README.md`.
- The scheme `stakeholder_objective_variable_scheme.png` was regenerated with an arrow from skirt depth (`x1`) to the citizens' objective (`INFLUENCES` in `objective_scheme.ipynb`). A new setting `VARIABLE_ORDER` in that notebook fixes the left-to-right order of the variables; without it the automatic layout swapped `x1` and `x2`.

### Consequences
- The citizens' objective now largely agrees with plastic removal instead of conflicting with it. It differs only by penalising high towing speeds and coarse meshes. The remaining conflicts are with cost, fishing-area interference and ecological risk.
- Today's design scores 21.8 for citizens instead of 14.7.
- minmax: still 3 systems, but faster (0.45 instead of 0.37 m/s) and with a slightly coarser mesh; lowest score ~51.
- a-fine: changes from 1 small, slow system (removal score 7) to 3 systems with the maximum spacing (removal score ~50). Both methods now give nearly the same design.

---

## 9. Preferences after the stakeholder game (sections *Weights*, *Preference Curves and Preference Functions*)

### Problems in the previous version
- The notebook ran `weights = weights_after` together with `prefs = prefs_before`, so the result belonged to neither the technical nor the social cycle.
- `pref_after` was an identical copy of `prefs_before`, so the social cycle only changed the weights.
- The ecological-risk curve in the code (1,000 / 50,000 / 100,000) differed from the one in the markdown table and in `Constants_and_Sources.ipynb` (2,000 / 25,000 / 250,000). The code version is the correct one.

### New version
- The two sets belong together like the weights: `weights_before` with `prefs_before` (technical cycle), `weights_after` with `pref_after` (social cycle). Both switches are now set to *after*; the comments next to them say which sets belong together.
- `pref_after` changes two curves. The weights carry the influence of a stakeholder, so a curve only changes where the stakeholder judges the outcome differently after seeing the technical-cycle design:

| Objective | `prefs_before` | `pref_after` | Reason |
|---|---|---|---|
| Ecological risk [animals/yr] | 1,000 / 50,000 / 100,000 | 1,000 / 25,000 / 60,000 | Ecological impact or bycatch is a criterion for four of the five stakeholders in the MCDA, but only the conservation advocates carry it in the optimisation, and their weight drops to 0.175. Halfway satisfied at about today's level, unacceptable at three times today. |
| Net fragmentation [t/yr] | -216 / -33 / 0 | -216 / -20 / 0 | The citizens' MCDA criteria (health, cleanliness) depend on visible progress; their influence is the lowest (0.125). Halfway satisfied at ~670 instead of ~1100 t/yr removed. |

- Cost, plastic removal and fishing-area interference are unchanged in `pref_after`.
- The markdown table now shows the ecological-risk curve of the code; a new subsection *Preferences after the stakeholder game* explains `pref_after`. The table *Preference curve - Ecological risk* in `Constants_and_Sources.ipynb` shows both versions.
- The values in `pref_after` are a proposal and judgement. Replace them if the stakeholder game gives other answers.

---

## 10. Constraint 1 and drag coefficient from measured data (sections *Constraints*, *Objective Functions*)

### Problem in the previous version
Today's design (`X_irl`: 4 m skirt, 1600 m span, 0.70 m/s) violated constraint 1: 964 kN per cable against an allowable 550 kN. The real System 03 is operated at about these settings (span 1,460 - 1,800 m, 0.75 m/s; EIA and The Ocean Cleanup, 2026), so the numbers of the constraint were wrong, not the design. Neither number had a usable source: 550 kN was a proxy for an unknown rope, and `Cd = 1.2` is the drag coefficient of a single twine, not of the system.

### New version

| Constant | Previous | New | Source |
|---|---|---|---|
| `Cd` (constraint 1) and `drag_coeff` (cost objective) | 1.2 | 0.7 | Effective coefficient on skirt depth x span, calibrated on 18 measured towline tensions of System 002 (Gonzalez Jimenez et al., 2023, Fig. 19). Least-squares fit: 0.72. |
| `max_tension` | 550 kN | 1677 kN | Bollard pull of one towing vessel, 171 t (Maersk Supply Service T-type specification sheet). |

- The formulas were not changed, only the two values and their comments.
- The same drag coefficient is used in constraint 1 and in the cost objective, so fuel cost drops: today's design costs 9.8 instead of 11.2 M€/yr, and the cost range is 7.7 - 116 instead of 7.7 - 143 M€/yr. The cost preference points (7.5 / 30 / 60) still correspond to about three and six systems at today's settings (3 x 9.8 and 6 x 9.8 M€/yr).
- The derivation, plausibility checks and limitations are in `Constants_and_Sources.ipynb` (new section *Calibration of the drag coefficient*, reference 22 added).

### Result
- Today's design reaches ~560 kN per cable, a third of the limit, and satisfies all three constraints.
- No design within the bounds exceeds the limit (at most ~970 kN), so constraint 1 is **not active** any more. Only constraint 3 limits the design (towing speeds above ~0.73 m/s); about 96 % of random designs are feasible.
- The towing speed of the optimum is no longer capped by the cable tension. Check runs outside the notebook (min-max): technical cycle 3 systems at ~0.53 m/s (before: 0.43 m/s), ~27.5 M€/yr and ~1,300 t/yr; social cycle 4 systems at ~0.45 - 0.55 m/s with a mesh near the upper bound of 20 mm, ~35 M€/yr, ~1,350 - 1,400 t/yr and ~42,000 animals/yr.
- The outputs stored in the notebook are from before these changes. Run all cells to update them.

---

## 11. Open points

1. Find sources for the new constants `extraction_interval`, `plastic_bulk_density` and `trip_duration`, and add them to `Constants_and_Sources.ipynb`.
2. Decide whether to keep constraints 1 and 2, since they no longer limit the design. Constraint 1 could become active again with a wider speed or depth range, or with a sourced strength of the towing line or net.
3. Document the meaning of the factor `0.50` in constraint 3.
4. Discuss the preference limits marked "judgement" in section 2 and the proposed `pref_after` (section 9) with the team, and replace them with the answers of the stakeholder game.
5. The budget limits of the cost curve (30 and 60 M€/yr) need a source.
6. Two of the five objectives (plastic removal and net fragmentation) pull in the same direction; mention this in the reflection.
7. The drag coefficient is calibrated on System 002 (10 mm mesh) and does not depend on the mesh size `x5`; the specification sheet used for the bollard pull is that of a sister vessel of the Maersk Tender.
8. The stored a-fine result is the best design of the first random generation (see section 7); the wrapper described there is not in the current optimisation cell.
