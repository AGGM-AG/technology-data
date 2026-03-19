# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`technology-data` is a data compilation pipeline that aggregates energy technology cost assumptions from 14+ sources (Danish Energy Agency, NREL ATB, Fraunhofer, EWG, PyPSA) into standardized CSVs for years 2020–2050. It is used by PyPSA-Eur and PyPSA-Earth energy system models.

## Environment Setup

Uses [Pixi](https://prefix.dev/) for environment management (preferred over Conda directly):

```bash
pixi install          # install all dependencies
pixi shell            # activate the environment
```

Alternatively, use the Conda environment:
```bash
conda env create -f environment.yaml
conda activate technology-data
```

## Common Commands

```bash
# Run unit tests
pytest test
# or
make unit-test

# Run integration tests (full Snakemake pipeline)
make test

# Run a single test file
pytest test/test_compile_cost_assumptions.py

# Run a single test by name
pytest test/test_compile_cost_assumptions.py::test_function_name

# Run the full pipeline via Snakemake
snakemake --cores all --configfile config.yaml

# Run only the global cost compilation rule
snakemake --cores all -f compile_cost_assumptions --configfile config.yaml

# Run only the US cost compilation rule
snakemake --cores all -f compile_cost_assumptions_usa --configfile config.yaml

# Lint and format (via ruff)
ruff check scripts/ test/
ruff format scripts/ test/
```

## Architecture

### Pipeline Flow

```
inputs/ (Excel, CSV, Parquet from DEA, NREL, Fraunhofer, EWG)
    └─▶ scripts/compile_cost_assumptions.py
            └─▶ outputs/costs_{year}.csv  (2020–2050, global)
                    └─▶ scripts/compile_cost_assumptions_usa.py
                            └─▶ outputs/US/costs_{year}.csv
```

### Key Files

- **`Snakefile`** — Orchestrates rules: `compile_cost_assumptions`, `compile_cost_assumptions_usa`, `convert_EWG`, `all`, `purge`
- **`config.yaml`** — Target years, EUR output year (2025), NREL ATB vintages, data source flags (PNNL, Vartiainen, ETIP, Fraunhofer), digit rounding precision
- **`scripts/compile_cost_assumptions.py`** — Core script (~4400 lines); reads all input sources, normalizes technology names/units/currency, applies inflation, outputs global CSVs
- **`scripts/compile_cost_assumptions_usa.py`** — US-specific layer; filters NREL ATB parquet files, maps to PyPSA names, merges US overrides
- **`scripts/_helpers.py`** — Shared utilities: `Dict` (attribute-access dict), `mock_snakemake()`, `prepare_inflation_rate()`, `adjust_for_inflation()`

### How Scripts Are Invoked

Both compilation scripts use Snakemake's `snakemake` object to receive inputs/outputs/config. For development/testing outside Snakemake, `mock_snakemake()` in `_helpers.py` constructs a mock object from `config.yaml`.

### Test Structure

- `test/conftest.py` — Session-scoped `config` fixture + function-scoped data fixtures (`cost_dataframe`, `atb_cost_dataframe`)
- Tests import directly from `scripts.compile_cost_assumptions` and `scripts.compile_cost_assumptions_usa`
- `test/test_data/` — Contains sample input data for unit tests

## Code Style

Enforced via `ruff.toml` (pyflakes, pycodestyle, isort, pydocstyle, pyupgrade) and pre-commit hooks. Run `pre-commit install` after cloning to enable hooks.
