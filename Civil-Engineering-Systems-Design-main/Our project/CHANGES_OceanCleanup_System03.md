# Changes to `OceanCleanup_System03.ipynb`

This file documents the changes made to the existing code of `OceanCleanup_System03.ipynb`, so the team can trace what was changed, why, and what is still open. Section names refer to the headings in the notebook.

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

## 5. Open points

1. Find sources for the new constants `extraction_interval`, `plastic_bulk_density` and `trip_duration`, and add them to `Constants_and_Sources.ipynb`.
2. Decide whether to keep constraint 2, since it never limits the design.
3. Document the meaning of the factor `0.50` in constraint 3.
4. Discuss the preference limits marked "judgement" in section 2 with the team, and update `pref_after` after the stakeholder game.
5. The `'a-fine'` run stops after 16 generations with the GA's own warning about fast convergence (it converges to a single-system design). Consider a larger `max_stall` or checking the result with a second run.
