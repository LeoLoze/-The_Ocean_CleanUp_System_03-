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

## 11. Skirt depth: underflow and bycatch depth factor (section *Objective Functions*)

### Problem in the previous version
- `plastic_depth = 5` m was not what its source says. 5 m is the depth range that Reisser et al. (2015) sampled; the depth scale they found for the plastic mass is 0.26 - 0.59 m. With such a value the depth term is practically 1, a deeper skirt has no benefit, and the skirt depth ends at its lower bound.
- The bycatch scaled linearly with the skirt depth. This contradicts the reasoning of objective 4, which uses the swept area instead of the swept volume because the bycatch is fish living near the surface: swept area times skirt depth is the swept volume again.

### New version

| Item | Previous | New | Source |
|---|---|---|---|
| Depth term of objective 2 | `1 - exp(-x1 / plastic_depth)`, `plastic_depth = 5` m: share of the plastic above the skirt depth | `1 - exp(-x1 / underflow_depth)`, `underflow_depth = 1.5` m: share of the plastic that is not dragged under the skirt | Assumption. Rough calibration on the retention efficiency of System 002/002A (3 m wings, 68 %) and System 002B (4 m wings, 73 %) in Sainte-Rose et al. (2026), Table 2. |
| Depth factor of objective 4 | `x1 / depth_ref` | `np.sqrt(x1 / depth_ref)` | The EIA expects the 4 m wings to catch fish only "to a slightly greater extent" than the 3 m wings, because most species in the bycatch "would not be affected" (CSA Ocean Sciences Inc., 2023, EIA Addendum §3.3.2.2). The square root gives +15 % for this step instead of +33 %. The exponent of 0.5 is an assumption. |

- The constant is renamed from `plastic_depth` to `underflow_depth`; the form of the formula in objective 2 is unchanged.
- `bycatch_ref` is unchanged, because both depth factors are 1 for the reference system (3 m).
- The derivations and limitations are in `Constants_and_Sources.ipynb` (new sections *Underflow depth scale* and *Depth factor of the bycatch*).

### Result
- Today's design (`X_irl`) removes ~770 instead of ~456 t/yr and catches ~17,000 instead of ~20,000 animals/yr; net fragmentation is -21.5 instead of -12.0 t/yr. The per-tonne cross-check of the bycatch rises to ~51,000 animals/yr.
- The reasoning texts of the ecological-risk curves were updated to the new level of today's design (e.g. 25,000 is now about 1.5 times today, 60,000 about 3.5 times). The curve values were not changed.
- Check runs outside the notebook (social cycle, 8 GA runs per method): the weighted sum of the preference scores is highest at a skirt depth of ~3.5 m, but it is nearly flat (64.8 at 3 m, 64.9 at 3.5 m, 64.2 at 6 m). Min-max gives 2 systems with a skirt of 3.7 - 5.3 m at the same score; a-fine gives mostly 1 system with a skirt of 3.3 - 5.1 m. Before these changes the skirt went to the upper bound of 6 m.
- The outputs stored in the notebook are from before these changes. Run all cells to update them.

---

## 12. One fuel model for the cost objective and constraint 3 (sections *Constraints*, *Objective Functions*, *Preference Curves and Preference Functions*)

### Problems in the previous version
- The cost objective and constraint 3 used two different fuel models. The cost objective took the fuel from the towing power only (about 2 M€/yr for today's design). Constraint 3 used `K_base` and `K_weight`, which depended on the towing speed only and had no documented derivation (factor `0.50`).
- Constraint 3 did not use the values of `Constants_and_Sources.ipynb` (`Fuel_max` was 1.25 instead of 2.4 million litres).
- The reported charter cost includes fuel, so fuel was counted twice in the cost objective.
- Several markdown texts and code comments no longer matched `Constants_and_Sources.ipynb` (e.g. the derivation of `bycatch_ref` and the sources of `v_critical`, `v_avoid`, `safety_buffer`, `plastic_size_char` and `frag_factor`).

### New version
- New function `fuel_per_day(x1, x2, x4)` in the objective cell, used by `objective_function_1` and by `constraint_3`: the fuel use measured on System 002, plus the fuel for the difference in towing power between the design and System 002 (3 m skirt, 520 m span, 0.75 m/s).

| Constant | Value | Source |
|---|---|---|
| `fuel_ref` | 18.1 t/day | 2 vessels x 9.05 t per day, average of campaigns 1 - 12 (CSA Ocean Sciences Inc., 2023, §2.4.1) |
| `span_ref` | 520 m | Span of System 002 in nominal mode (CSA Ocean Sciences Inc., 2023) |
| `sfoc` | 0.175 kg/kWh | Medium-speed diesel engine on marine diesel oil (IMO, 2020, Table 19) |
| `fuel_price` | 1,050 EUR/t | Sainte-Rose et al. (2026), Supplementary Table S6 |
| `litres_per_tonne` | 1,185 L/t | Gas/diesel oil (IEA, 2005, Table A3.8) |
| `charter_cost` | 7.2 M€/yr (was 13.2) | 13.13 M€ reported including fuel (The Ocean Cleanup, 2025) minus 5.9 M€ fuel of today's design |

- Removed: `fuel_cost`, `K_base` and `K_weight`.
- Constraint 3 now uses `Fuel_max = 2.4e6` L and `trip_duration = 49` days, as in `Constants_and_Sources.ipynb`, and depends on skirt depth, vessel spacing and towing speed.
- The extraction interval is not part of the fuel model: an extraction takes less than 12 hours and happens at most every 7 days, and the measured fuel use is a campaign average that already includes it.
- The ecological-risk curves were multiplied by 2.9, like `bycatch_ref`: 2,900 / 145,000 / 290,000 before the game and 2,900 / 72,500 / 174,000 after. The scores and the reasoning stay as they were.
- Markdown and comments corrected to `Constants_and_Sources.ipynb`: constraint table and text, cost objective, derivation of `bycatch_ref`, the cost and ecological-risk rows of the preference tables, and the sources in the constants cell.
- `Constants_and_Sources.ipynb`: the `charter_cost` row is updated, the rows of `fuel_cost`, `K_base` and `K_weight` are replaced by the rows of the new constants, and references 25 and 26 are added.

### Result
- Fuel use: 18.1 t/day for System 002, about 22 t/day for today's design and 16.6 - 26.6 t/day within the bounds. Only about a tenth of the measured fuel use is towing work.
- Today's design costs 13.4 M€/yr (7.2 charter + 0.3 material + 5.9 fuel), which matches the reported 13.13 M€; one system costs 11.9 - 14.7 M€/yr. The cost points of 30 and 60 M€/yr now correspond to a little more than two systems and to between four and five systems at today's settings. The curve values were not changed.
- No constraint is active any more. Today's design uses 53 % of the fuel capacity in one campaign and no design within the bounds uses more than 64 %, so the towing speed is no longer capped at ~0.73 m/s.
- Check runs outside the notebook (5 GA runs per method). Social cycle: min-max gives 2 systems at ~0.74 m/s with a mesh of 18 - 20 mm (~28 M€/yr, ~1,870 t/yr, ~99,000 animals/yr); a-fine gives 1 system at 0.66 - 0.73 m/s (~13 M€/yr, 740 - 910 t/yr). Technical cycle: min-max gives 2 systems at 0.58 - 0.64 m/s with a 10 mm mesh (~25.5 M€/yr, ~1,700 t/yr, ~118,000 animals/yr).
- The outputs stored in the notebook are from before these changes. Run all cells to update them.

---

## 13. Clean-up of comments, markdown and references (both notebooks)

No value, formula, bound, weight or preference point was changed. The executable code of `OceanCleanup_System03.ipynb` is identical before and after, so the stored outputs are still valid.

### `OceanCleanup_System03.ipynb`
- Code comments no longer name sources; the sources are only in `Constants_and_Sources.ipynb`. The descriptions in the comments follow that document.
- Three comments that contradicted the model were corrected: the skirt-depth bound (underflow instead of the natural depth of the plastic), "800 is the size of System 02" (800 m is its barrier length; its span is 520 m) and "width of the net 2 km" (the barrier is 2.2 km long).
- The "placeholder" notes, the template instructions and the closing note were removed, and the markdown was shortened from about 23,000 to 15,000 characters. All numbers in the markdown were recomputed with the current model.
- The claim that the speed term "gives an optimal towing speed" was removed: the product of speed and retention peaks at 0.99 m/s, above the speed bound of 0.75 m/s.
- The bounds and today's design are now in the variable table; the weights of both cycles are in the weights table.
- The image link no longer goes through the parent folder (`Stakeholder_Objectives_and_optimisation/...png`).

### `Constants_and_Sources.ipynb`
- The reference list is sorted alphabetically, written in one format, and contains exactly the sources that are cited (17 entries).
- Removed because nothing cites them any more: System 002 mid-term evaluation, Liese et al. (2026), the $7.5 billion press release, Reisser et al. (2015), the Great Pacific Garbage Patch page, the System 002 milestone page, "System 002 and Marine Life" and the neuston FAQ. Removed as duplicates: the second entries of the Beginner's Guide and of the EIA, and "Ships and Oil".
- Added because they were cited without an entry: United Nations (1982) and the Maersk Topper sheet (source of the 171 t bollard pull). Lebreton et al. "(2019)" for the garbage-patch paper is corrected to 2018.
- Citation keys now match the list (e.g. Leal Filho et al., Saettone et al., Maersk Supply Service n.d.-a / n.d.-b). `uptime` cites Sainte-Rose et al. (2026), which is the paper the old entry "The Ocean Cleanup (2026)" pointed to. `rho_water` is marked as a standard value instead of "Wikipedia".
- Notes corrected: `escape_steepness` (the retention is 0.5 at `v_critical`, not 1.0 below it), `bycatch_ref` (66 &times; 0.044, not 0.44), `extraction_interval` ("5.5 days is average" removed, it does not follow from 2.5 to 7 days). `fuel_max` is renamed to `Fuel_max`, as in the code. Heading of constraint 2 renamed to *Maximum retention zone capacity*.

---

## 14. Sourced bounds and today's design, one vessel sheet, third scenario (sections *Design Variables and Bounds*, *Constraints*, *Weights*, *Preference Curves and Preference Functions*)

The optimisation cell (GA call and options) and `genetic_algorithm_pfm` were not changed.

### Bounds and today's design

| Variable | Previous bounds | New bounds | Previous `X_irl` | New `X_irl` | Source |
|---|---|---|---|---|---|
| `x1` skirt depth [m] | 3 - 6 | 3 - 6 | 4 | 4 | 3 m = System 002, 4 m = System 03 (CSA Ocean Sciences Inc., 2023, Table 2-1). 6 m is an assumption. |
| `x2` vessel spacing [m] | 800 - 1600 | 520 - 1800 | 1600 | 1460 | 520 m = span of System 002, 1,460 m = span of System 03 (CSA Ocean Sciences Inc., 2023); 1,800 m = span simulated for the next systems (Sainte-Rose et al., 2026, Table 1) |
| `x3` number of systems [-] | 1 - 10 | 1 - 20 | 1 | 1 | Largest fleet in the cleanup scenarios (Sainte-Rose et al., 2026, Table 1) |
| `x4` towing speed [m/s] | 0.3 - 0.75 | 0.26 - 1.29 | 0.70 | 0.75 | 0.5 to 2.5 knots (CSA Ocean Sciences Inc., 2023); 0.75 m/s operating speed (Sainte-Rose et al., 2026, Table 1) |
| `x5` mesh size [mm] | 10 - 20 | 10 - 20 | 16 | 16 | 10 mm = System 002, 16 mm = System 03 (CSA Ocean Sciences Inc., 2023, Table 2-1). 20 mm is an assumption. |

- New table *Design variables - bounds and today's design* in `Constants_and_Sources.ipynb`.

### One vessel sheet
- `max_tension` is 1700 kN (was 1677): 173.3 t bollard pull on the sheet of the Maersk Traveller, the sheet that already gave the fuel capacity (2 x 1,191 m3). The Maersk Topper sheet (171 t) is no longer used.
- No sheet is available for the Maersk Tender and Maersk Trader, which tow System 03. They are vessels of the same class and length (73.2 m) as the Maersk Traveller. The two sister sheets we read give 171 t and 173.3 t bollard pull, but 1,960 and 1,191 m3 of fuel, so the fuel capacity used is the lower one.

### Constant that follows from today's design
- `charter_cost` is 7.1e6 (was 7.2e6): the reported 13.13 M€ minus the fuel of today's design, which is now 22.6 t/day (6.1 M€) instead of 22 t/day (5.9 M€).

### Third weight and preference set
- `weights_env = [0.15, 0.20, 0.15, 0.35, 0.15]`: the scenario *Environmental priority* of the MCDA. The three weight sets are now the three stakeholder-weight scenarios of the MCDA.
- `pref_env`: as `pref_after`, but with a stricter ecological-risk curve, 2,900 / 50,000 / 100,000 (halfway satisfied at today's bycatch, which the EIA rated *Low*; unacceptable at twice today). This is a proposal, like `pref_after`.
- The switches `weights` and `prefs` still select one cycle per run; they are left on the social cycle.

### Result
- Today's design: 13.4 M€/yr, ~860 t/yr, 4.5 km2, ~51,000 animals/yr, -24.9 t/yr.
- The constraints are active again: about 5 % of the designs within the bounds exceed the cable tension and about 7 % the fuel capacity (deep skirt, wide spacing and high speed together). The retention zone capacity is only reached at the very edge (25.1 t at 6 m, 1,800 m, 0.99 m/s). Today's design uses 35 %, 66 % and 55 % of the three limits.
- The plastic removed per hour is now highest at about 1.0 m/s, inside the speed bounds.
- Check runs outside the notebook with the unchanged GA settings (min-max, 4 runs each; no Tetra):

| Cycle | Systems | Skirt [m] | Spacing [m] | Speed [m/s] | Mesh [mm] | Cost [M€/yr] | Removal [t/yr] | Bycatch [animals/yr] |
|---|---|---|---|---|---|---|---|---|
| Technical | 2 | 3.0 - 3.8 | 1,690 - 1,800 | 0.58 - 0.61 | 10 | 25 | ~1,760 | ~114,000 |
| Social | 2 | 3.8 - 5.5 | ~1,800 | 0.64 - 0.74 | 19 - 20 | 27 | ~1,990 | ~94,000 |
| Social, environmental priority | 3 | 3.2 - 4.7 | 1,640 - 1,800 | 0.29 - 0.31 | 18 - 20 | 36 | ~1,300 | ~27,500 |

- a-fine gives one system in all three cycles, with different settings in every run.
- **The outputs stored in the notebook are from before these changes. Run all cells to update them** (once per cycle, with `weights` and `prefs` set to that cycle).

---

## 15. Retention zone counted over towing days, results section, plot setting (sections *Importing Required Packages*, *Constraints*, *Optimisation*, new section *Results of all cycles*)

The optimisation cell (GA call and options) and `genetic_algorithm_pfm` were not changed.

### Constraint 2 – retention zone capacity

| | Previous | New |
|---|---|---|
| Plastic in the retention zone at extraction | `removed_per_system * (extraction_interval / 365)` | `removed_per_system / (uptime * 365) * extraction_interval` |

- The previous version filled the retention zone at the yearly-average rate, which includes the 30 % of the year in which the system does not tow. A system only collects while it tows, so the 7 days between two extractions are now towing days.
- Effect: one system can remove at most 25 t per 7 towing days, about 910 t/yr. Today's design fills 95 % of the retention zone instead of 66 %. About 17 % of the designs within the bounds exceed the limit (before: none), and the two-system optima of the technical and social cycle sit at the limit.
- In the social cycle the min-max score is now set by The Ocean Cleanup alone (two systems at the limit give a score of 61). Designs with a bycatch between about 75,000 and 102,000 animals/yr have the same min-max score, so the GA returns a different design in every run. The results table holds two runs.

### Results section (new cells after the optimisation cell)
- One markdown cell and one code cell. The code cell holds the designs of all runs (three cycles with min-max and Tetra, a repeat run of the social cycle, the two runs with the weights and curves of different cycles, and a run with the donor budget limits doubled) and evaluates them with the model: objective values, cost and bycatch per tonne, preference scores, min-max score and use of the three constraint limits.
- Two checks are printed: the most plastic that one system can remove within the constraints, and the share of the improvement that a second system gives.
- The designs are the values printed by the optimisation cell (rounded to two decimals). After a new run, copy the printed design into its row.
- New import in that cell: `pandas`.

### Smaller changes
- Import cell: `axes.formatter.limits` and `axes.formatter.use_mathtext`, so that the axis numbers of the bycatch panel no longer overlap in both figures.
- *Optimisation* markdown: one sentence on the convergence warning of a-fine.
- *Preference Curves* markdown: the changes in `pref_after` and `pref_env` are the outcome of our stakeholder game (was: "our proposal").
- `Constants_and_Sources.ipynb`: `extraction_interval` is described as towing days.
- `Stakeholder_Objectives_and_optimisation/README.md` rewritten (it still described the project before the model existed). `OLD_MCDA_OceanCleanup_System03.ipynb` removed.
- The notebook was run once after these changes (social cycle); the stored outputs belong to the current code. `Summary/System03_Optimisation_Summary.docx` and its figures were rebuilt from the results table.

---

## 16. Retention factor: target and measured retention (sections *Objective Functions*, *Results of all cycles*)

The optimisation cell (GA call and options) and `genetic_algorithm_pfm` were not changed.

### Problem
The model retains 71 % of the plastic that today's design meets (93 % not lost under the skirt x 90 % not lost to speed x 85 % larger than the mesh). This matches System 002 and 002B (68 and 73 %), on which `underflow_depth` was calibrated, but not System 03, for which 40 % was measured (Sainte-Rose et al., 2026, Table 2). The paper uses 40 % as today's performance and 70 % as the improvement target.

### New version
- New constant `retention_factor = 1.0` (constants cell, objective 2). 1.0 is the target retention, the model as it was. 0.56 is the measured retention: 40 % / 71 %.
- `objective_function_2` multiplies its result by `retention_factor`. Through that function the factor also reaches the avoided microplastic in objective 5 and the retention zone constraint. Nothing else changes: bounds, `X_irl`, weights, preference curves and all other constants are as before.
- One sentence in the description of objective 2; one row in `Constants_and_Sources.ipynb`.
- Results cell: four rows with the measured retention (today's design and the min-max optimum of each cycle). Every row is evaluated with its own factor; rows without a factor use 1.0. The cell sets `retention_factor` while it evaluates and puts it back to the value of the constants cell at the end.
- You moved `import pandas as pd` from the results cell to the import cell; this is kept.

### Result with the measured retention (factor 0.56)

| | Target (1.0) | Measured (0.56) |
|---|---|---|
| Today's design: plastic removal | 863 t/yr | 483 t/yr |
| Today's design: cost per tonne | 15.6 k€ | 27.8 k€ |
| Today's design: score of The Ocean Cleanup | 44 | 26 |
| Today's design: retention zone filled | 95 % | 53 % |
| Min-max optimum, technical / social / environmental | 2 / 2 / 3 systems | 2 / 3 / 3 systems |

- With the measured retention one system removes at most about 720 t/yr, below the 910 t/yr that the retention zone allows, so that limit no longer binds.
- The notebook was run once after these changes with the default of 1.0 (social cycle). The three measured-retention designs come from runs of the notebook with `retention_factor = 0.56` and the switches of each cycle (min-max). The Word summary was rebuilt.

---

## 17. Open points

1. The retention zone limit rests on the choice of 7 towing days between two extractions (the EIA gives 2.5 to 7 days). With a shorter interval it would not bind.
2. The measured-retention case scales the removal with one factor; it does not explain why the long wings of System 03 retain less than System 002.
3. The budget limits of the cost curve (30 and 60 M€/yr) need a source. They were set when one system cost ~9.8 M€/yr; it now costs ~13.4 M€/yr.
4. Two of the five objectives (plastic removal and net fragmentation) pull in the same direction; mention this in the reflection.
5. The drag coefficient is calibrated on System 002 (10 mm mesh, 0.50 - 0.86 m/s) and does not depend on the mesh size `x5`; the speed bounds now go up to 1.29 m/s. The specification sheet used for the bollard pull and the fuel capacity is that of a sister vessel of the Maersk Tender.
6. The stored a-fine result is the best design of the first random generation (see section 7); the wrapper described there is not in the current optimisation cell.
7. `underflow_depth` (1.5 m) and the square-root depth factor of the bycatch are assumptions (section 11). The optimum is nearly flat in the skirt depth, so the GA result for `x1` differs from run to run.
8. The plastic-removal and net-fragmentation curves were set when today's design removed ~456 t/yr; it now removes ~860 t/yr.
9. The fuel model (section 12) assumes that everything except the towing work, about 90 % of the measured fuel use, does not depend on the design. The measured value is a campaign average of System 002 that includes transit.
