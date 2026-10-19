# Copilot instructions

## Build, test, and lint

Run commands from the repository root.

- Install the package in editable mode with test dependencies:
  `python -m pip install -e ".[tests]"`
- Run the checked-in tests in the main CI test areas (CI also checks the
  separately cloned `pyomo-model-libraries` tree):
  `python -m pytest -v -W ignore::Warning pyomo examples doc`
- Run one test, or one test class:
  `python -m pytest pyomo/core/tests/unit/test_component.py::TestComponent::test_getname`
  `python -m pytest pyomo/core/tests/unit/test_component.py::TestComponent`
- Check Python formatting (CI):
  `black . --check --diff`
- Run the configured spelling check (CI uses `typos`):
  `typos --config .github/workflows/typos.toml`

The root `conftest.py` selects tests marked `default` when no marker expression
is given. Solver- and writer-specific tests can be selected with
`--solver SOLVER` or `--writer WRITER`; pytest marker expressions can also
select these tests, for example `-m "solver(vendor='gurobi')"`.

For documentation, install its dependencies with
`python -m pip install -e ".[docs]"`, then build or test with:

- `make -C doc/OnlineDocs html`
- `make -C doc/OnlineDocs doctest`

## Architecture

Pyomo is a modeling framework: users build symbolic optimization models in
Python, then use solver interfaces to translate and solve those models.

- `pyomo.core` provides modeling components (such as sets, parameters,
  variables, constraints, objectives, and blocks), expression trees, and the
  kernel API. `pyomo.environ` is the main public import surface: it loads
  plugin packages and re-exports commonly used modeling and solver APIs.
- `pyomo.opt` defines solver factories, problem/result formats, and solver
  manager interfaces. Solver implementations and conversion plugins live in
  `pyomo.solvers`; model writers and representation-generation code are
  primarily in `pyomo.repn`.
- Specialized modeling features are organized in focused packages such as
  `pyomo.gdp`, `pyomo.dae`, `pyomo.mpec`, and `pyomo.network`. `pyomo.dataportal`
  handles model data input, while `pyomo.scripting` provides command-line
  entry points.
- `pyomo.common` contains shared Pyomo utilities and is deliberately
  independent of the rest of the `pyomo` package. `pyomo.contrib` contains
  optional or separately deployable extensions.

## Repository-specific conventions

- Tests are distributed with their corresponding package areas under
  `pyomo/**/tests`; project-wide pytest behavior is configured in the root
  `conftest.py` and `pyproject.toml`. Tests requiring particular solvers or
  writers use the `solver` and `writer` markers, while `default`, `expensive`,
  `mpi`, `neos`, and `importtest` are also registered markers.
- Plugin discovery is explicit. `pyomo.environ` imports the `.plugins` modules
  for registered packages and calls their `load()` functions. When adding a
  contrib package that defines plugins, update the package list in
  `pyomo/environ/__init__.py` so the plugins are loaded through the public
  environment import.
- The Black configuration is in `pyproject.toml` (88-character line length,
  with project-specific exclusions). The CI spelling configuration is
  `.github/workflows/typos.toml`.
- Contribution requirements and license terms are in `CONTRIBUTING.md`;
  detailed contribution guidance is linked there to the online documentation.
