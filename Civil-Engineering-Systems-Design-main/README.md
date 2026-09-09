# Systems Design - Optimization Model

A genetic algorithm-based framework for multi-objective optimization with preference function modeling.

## Overview

This project implements a genetic algorithm for optimizing multi-objective problems, using different aggregation methods (affine and min-max). Additionally, a Tetra aggregation method is implemented, but it is **not required** for the CIEM000 Unit 1 assignment.

## Project Structure
```
genetic_algorithm_pfm/          # Main package
├── algorithm.py                # Core genetic algorithm implementation
├── ...
example notebooks (use these!)
```

## Usage

Direct interaction with the genetic algorithm source code is not required to use this project. Optimization models are run via the example notebooks provided. These notebooks are intended to serve as templates and may be copied and adapted to define new optimization problems.

## Getting Started

### 1. Maintaining the Directory Structure

The notebooks rely on relative imports, meaning the `genetic_algorithm_pfm` package must remain in the same directory as the notebook from which it is called. Moving a notebook without relocating the corresponding package will result in a: `ModuleNotFoundError: No module named 'genetic_algorithm_pfm'`.


### 2. Setting Up the Environment

A Python environment containing the required dependencies must be configured prior to running the notebooks. For those unfamiliar with this process, a virtual environment provides an isolated workspace for a project's dependencies, preventing conflicts with other Python projects on the same system.

A complete introduction to environment setup is available in the [MUDE book](https://mude.citg.tudelft.nl/book/2025/_git/github.com_TeachBooks_learn-programming/mude-2025/book/environments.html). The `mude-base` environment established there already satisfies the dependencies listed below, and no additional installation is required for students using this environment.


## Examples

- `EXAMPLE_Artificial_Reef.ipynb` — A complete worked example demonstrating artificial reef design optimization, with detailed explanation of each step. Recommended as an introductory reference.
- `EXAMPLE_tbd (Li).ipynb` — A simplified notebook intended for use as a template for custom models.

## Documentation

- `aggregations.md` — Detailed documentation of the aggregation methods.
- `requirements.txt` — List of project dependencies.
