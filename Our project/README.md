# Our project – System 03 stakeholder objectives and optimisation

This folder holds our own work for the System 03 design assignment. The course material (genetic algorithm, reef example) lives one level up in `Civil-Engineering-Systems-Design-main/`.

## Structure

```
-The_Ocean_CleanUp_System_03-/
├── Our project/
│   ├── Ocean_Cleanup_Stakeholder_Matrix.xlsx        1. stakeholders, criteria, weights, alternative scores
│   ├── Stakeholder_Objectives_and_optimisation/
│   │   ├── objective_scheme.ipynb                   2. draws the scheme (stakeholder -> objective -> variables)
│   │   └── stakeholder_objective_variable_scheme.png    the resulting figure
│   └── README.md                                    this file
└── Civil-Engineering-Systems-Design-main/
    ├── OceanCleanup_System03.ipynb                  3. the optimisation notebook (our model)
    ├── EXAMPLE_Artificial_Reef.ipynb                   reference example we follow step by step
    └── genetic_algorithm_pfm/                          GA code provided by the course (do not edit)
```

## Workflow (how the pieces connect)

1. **Excel matrix** – who the stakeholders are, what they care about, and how much (weights).
2. **Scheme** – one objective per stakeholder, and the design variables that drive each objective.
3. **Optimisation notebook** – the variables become code; next come constraints, objective functions, preference functions and the GA run.

## The scheme (objectives and variables)

| Stakeholder | Objective | Influenced by |
|---|---|---|
| Investors / donors | Cost | x1, x2, x3, x4 |
| The Ocean Cleanup | Plastic removal | x1, x2, x3, x4 |
| North Pacific fishers | Fishing-ground access | x2, x3 |
| Marine conservation advocates | Ecological impact | x1, x3, x4, x5 |
| Citizens | Microplastic removal (health proxy) | x2, x3, x4, x5 |

| Variable | Description | Unit | Type |
|---|---|---|---|
| x1 | Skirt depth below waterline | m | continuous |
| x2 | Distance between support vessels | m | continuous |
| x3 | Number of systems deployed | – | integer |
| x4 | Towing speed | m/s | continuous |
| x5 | Mesh / screen size | mm | continuous |

The barrier length is **fixed** (2,500 m), not a variable. All bounds and the reference design `X_irl` are placeholders until we have sources.

## How to use the notebooks

**`objective_scheme.ipynb`** (only needs `matplotlib`)
1. Open it and choose the Python kernel. Run all cells.
2. The figure appears in the notebook and is saved as `stakeholder_objective_variable_scheme.png` next to it.
3. To change the scheme, edit the *Content* cell (stakeholders, objectives, variables, `INFLUENCES`) and run all cells again. The order of the variables is re-optimised automatically and the labels x1–x5 are renumbered left to right; the printed `labels:` line shows which variable got which number.

**`OceanCleanup_System03.ipynb`** (needs `matplotlib`, `numpy`, `scipy`)
1. Keep it inside `Civil-Engineering-Systems-Design-main/`. It imports `genetic_algorithm_pfm` by relative import and fails with `ModuleNotFoundError` anywhere else.
2. Run the cells from top to bottom. The design variables (x1–x5), their bounds, the fixed barrier length and the integer setting `var_type_mixed` are already in place.
3. The remaining sections follow `EXAMPLE_Artificial_Reef.ipynb`: constraints, objective functions (one per row of the table above), preference functions, then the optimisation.
4. Pass `var_type_mixed` to the GA `options` so that the number of systems (x3) stays a whole number.
5. If the scheme is changed and the variables get renumbered, update `design_variables`, `bounds`, `var_type_mixed` and `X_irl` in this notebook to the same order.

## Status and open points

- Done: stakeholders, objectives, variables, scheme, variable definitions in the optimisation notebook.
- To do: constraints, objective functions, preference functions, GA run and results.
- Placeholders to replace with sourced values: bounds (especially towing speed and mesh size), `X_irl`, barrier length (2,500 m from The Ocean Cleanup's 2024 update; the Excel notes say 2.2 km).
- The two sheets in the Excel file disagree on some weights (investors, citizens). Agree which one is authoritative before writing the preference functions.

Please work on a separate git branch and merge after checking, as described in the main README.
