# System 03 – stakeholder objectives and optimisation

Our work for the System 03 design assignment is in the folder above this one (`Our project/`).

| File | Content |
|---|---|
| `OceanCleanup_System03.ipynb` | The optimisation model: design variables, constraints, objectives, weights, preference curves, the optimisation and the results of all cycles |
| `Constants_and_Sources.ipynb` | Value, source and note for every constant, for the bounds and for today's design; reference list |
| `MCDA_OceanCleanup.ipynb` | MCDA of the design alternatives; its three stakeholder-weight scenarios are the weights of the optimisation |
| `Ocean_Cleanup_Stakeholder_Matrix.xlsx` | Stakeholders, criteria, weights and scores of the alternatives |
| `Stakeholder_Objectives_and_optimisation/objective_scheme.ipynb` | Draws the scheme below (`stakeholder_objective_variable_scheme.png`) |
| `Summary/` | Summary of the optimisation results, with the figures |
| `CHANGES_OceanCleanup_System03.md` | Log of the changes to the optimisation notebook |
| `genetic_algorithm_pfm/` | Genetic algorithm provided by the course (not edited) |

## The scheme (objectives and variables)

| Stakeholder | Objective | Influenced by |
|---|---|---|
| Investors / donors | Annual cost | x1, x2, x3, x4 |
| The Ocean Cleanup | Plastic removal | x1, x2, x3, x4, x5 |
| North Pacific commercial fishers | Fishing-area interference | x2, x3 |
| Marine conservation advocates | Ecological risk (bycatch) | x1, x2, x3, x4, x5 |
| Citizens | Future plastic fragmentation (net) | x1, x2, x3, x4, x5 |

| Variable | Description | Unit | Type |
|---|---|---|---|
| x1 | Skirt depth below waterline | m | continuous |
| x2 | Distance between support vessels | m | continuous |
| x3 | Number of systems deployed | – | integer |
| x4 | Towing speed | m/s | continuous |
| x5 | Mesh / screen size | mm | continuous |

The barrier length is fixed (2,200 m). The bounds and today's design are in `Constants_and_Sources.ipynb`.

## How to use the notebooks

**`OceanCleanup_System03.ipynb`** (needs `matplotlib`, `numpy`, `scipy`, `pandas`)
- Keep it in `Our project/`: it imports `genetic_algorithm_pfm` from the same folder.
- The optimisation runs one cycle per run. Set `weights` and `prefs` to the same cycle and run all cells. `'tetra'` needs an internet connection.
- The last cell holds the designs of all runs. After a new run, copy the printed design into its row.

**`objective_scheme.ipynb`** (needs `matplotlib`)
- Run all cells; the figure is saved as `stakeholder_objective_variable_scheme.png`.
- To change the scheme, edit the *Content* cell and run all cells again. `VARIABLE_ORDER` fixes the order of the variables, so that the labels x1–x5 match the optimisation notebook.
