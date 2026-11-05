# daqpyutils

_"daq-cmake, but for Python rather than C++"_

* Streamlines the Python development process
* Provides an executable script automating creation of Python-only repositories
* Defines our Python development standards

## `create-python-dunedaq-package`

Very much analogous to how the [`create_dunedaq_package`
script](https://dune-daq-sw.readthedocs.io/en/latest/packages/daq-cmake/#the-create_dunedaq_package-script)
automatically generates skeletons of DUNE DAQ standards-conformant C++
packages, `create-python-dunedaq-package` does the same for Python
packages. Along with taking the name of the desired package, it takes
a series of arguments which allow the developer to specify various
aspects of the package to generate:
```
create-python-dunedaq-package (various options) <package name>
```

...where these options will be described in a moment. It is recommended that you run
this script from the `pythoncode` subdirectory in your work area, as
this is where `dbt-build` expects the Python-only packages

Options include:
```
  -l, --log-level [critical|error|warning|info|debug|notset]
                                  Set the log level.
  -r, --requirement TEXT          Add a required package in the pyproject.toml
                                  (e.g. 'click')
  -rf, --requirements-file TEXT   Define multiple required packages in the
                                  pyproject.toml by pointing to a requirements
                                  file (e.g. requirements.txt).
  -a, --app-tuple TEXT            Create template scripts for applications to
                                  become available when installed
  -af, --app-file TEXT            Define applications for the pyproject.toml
                                  by pointing to a file containing a list of
                                  application names (e.g. applications.txt).
  -p, --package-description TEXT  Package description for the pyproject.toml.
  -s, --strict                    Strict mode. Adds extra tooling and constraints
      				  for the power developer.
  -h, --help                      Show this message and exit.
```

Note that the options `-r/--requirements` and `-a/--app` can be used
multiple times, e.g. `create-python-dunedaq-package -r click -r rich
<package_name>`

## An Overview of Our Package Layout

Once you've created a skeleton for your repository, before you start
developing you'll want to understand what the DUNE-DAQ development standards are for Python packages. This understanding can
be broken into two parts; one, knowing what the repository layout is and where to put different types of
code, and two, knowing how to develop the code via edits to
`pyproject.toml` and the tools it defines. Please read this [full description of the Python repository layout], which will describe the following files and directories besides `pyproject.toml`, which gets its own section later on this page:
```
<package_name>/
├── .git/
├── .github/
├── docs
│   └── README.md
├── scripts/
├── src/
│   └── <package_name>/
│       ├── apps/
│       │   └── __init__.py (empty)
│       └── integtest/
├── tests/
└── pyproject.toml
```
This structure is set up in order to simplify the installation and maintenance of DUNE-DAQ Python-only packages.

## `pyproject.toml`
This file defines the blueprint of how the package is intended to be build, what it requires in various installation modes, entry points, and standard tooling configuration. Each of these topics will be defined individually as follows.

Notes for the developer 
 - Should any changes be made in the `pyproject.toml`, a fresh `pip install` will be required to implement the changes.
 - It is recommended that when developing, the edit mode is used as `pip install -e <path_to_project_root>`

### Project specification and installation 
The current building tooling and versioning requirement is defined in `[build-system]`, which is configured to use `setuptools`. This is not expected to change on a per repository basis, and proposed changes with their justification should be discussed at a SWIT meeting. The developer is not expected to change these parameters. The other build system parameters specified are
 - `[tool.setuptools.packages.find]` - location of the project source code, not expected to change.
 - `[project.setuptools.package-data]` - location of project data files that should be placed inside the project structure. Commonly used examples are `html`, `json`, and `.pyi` files. This is required as otherwise these files are not packaged with the environment when using a virtual environment deployment.


The project that the repository implements has metadata defined in the `[project]` section, and project configuration parameters specified in dot scoped blocks. The base `[project]` specification including the following parameters
 - `name` - the name of the repository, expected to be static.
 - `description` - a description of the role of the repository, expected to be static.
 - `version` - the version number, which changes following the release schedule.
 - `readme` - the path to the base `README.md`, which provides a high-level overview of the usage of the repository, expected to be static.
 - `requires-python` - python version required to use this repository, expected to be static. As per the `[build-system]`. the python version is only excpected to change globally and not on a per-repository case.
 - `dependencies` - the list of packages required to use the repository source code.

The `project` may also specify optional dependencies, which are those tailored to the use case and do not form part of the source code dependencies. The common optional dependency groups and their dependencies included in the template include
 - `dev` - the additional packages a developer will commonly need, has packages:
   - `pytest` - runs unit/integration tests.
   - `pytest-cov` - calculates the coverage of the unit tests.
   - `ruff` - linting configuration (see later in this section).
   - `pre-commit` - defines a hook to execute the linting on `git commit`.
 - `test` - the additional packages a testing environment will commonly need, has packages:
   - `pytest`
   - `pytest-cov`
These optional installation modes can then be used as a suffix to the installation location as
```python
pip install <path_to_project_root>[mode]
```

The last component that is specified in `project` are the CLI entry points which are accessible if the package has been installed in the virtual environment, regardless of the source code location. These are specified in the `[project.scripts]` block, which typically point to individual applications living in the `<package_name>/src/<package_name>/apps/` path. 

### External tool configurations

Outside of specifying the installation tool and the project structure, the `pyproject.toml` also contains the configuration of external tools that are used within the repository. The default ones specified in the template are defined as follows

#### Ruff 
Ruff is an extremely fast Python linter and code formatter, designed as a modern, high-performance replacement for older tools like `Flake8`, `isort`, and `Black`. The `google` documentation docstring convention has been chosen for its readability with both IDEs and terminals, as well as using clear sections for argument, return type, and raise descriptors to give the reader an easy and quick understanding of the purpose of the code. The default implementation of `ruff` has the following linting rules

Rules:
 - `D` - `pydocstyle`: check for docstring conventions
 - `E` - `pycodestyle` errors: standard PEP 8 errors
 - `F` - `Pyflakes`: check for common programming errors
 - `I` - `isort`: best practices for import order and sorting
 - `UP` - `pyupgrade`: suggestions for code modernization
 - `RUF` - `ruff`-specific rules: built-in Ruff-specific warnings
 - `ANN` - `flake8-annotations`: check for missing type annotations

Ignore rules:
 - `D100` - Documentation existence - Every package needs a summary at the top of the module/file.
 - `D104` - Documentation formatting - Separate the summary from details.
 - `D212` - Documentation formatting - Docstring starts on the same line as `"""`.
 - `N999` - Naming linting - Package name follows PEP-8 compliance. The organization uses a different standard, mostly defined by Spack restrictions.

Select rules:
 - `D213` - Documentatino formatting - Docstring starts on new line after `"""`.

There are also specific rules implemented for the unit tests, these are defined in the `[tool.ruff.lint.per-file-ignores]` section. The default rules in the template include those in the ignore rules section above, and are extended to include
 - `D103` - allow public functions without docstrings
 - `S101` - allows for `assert`s in the tests
 - `ANN201` - allows for missing return types

## Strict mode and `mypy`
For most development in python, the implementation defined above is sufficient for safe development practices. However, for applications that are used very commonly, it is worth considering using the `--strict` operator to create the package. This introduces `mypy`, which is a signature checking tool that validates types used in function signatures and calls. In strict mode, the `mypy` tool is configured to use the following rules
 - Function and Definition Strictness
    - `disallow_untyped_defs = true` -  Force all functions to have type annotations for arguments and return values.
    - `disallow_incomplete_defs = true` -  Disallow functions that have some, but not all, arguments annotated.
    - `check_untyped_defs = true` -  Type-check the body of functions even if they lack type annotations.
    - `no_implicit_optional = true` -  Force 'Optional[T]' instead of just 'T' when a default value is None.
 - Warning and Code
    - `warn_return_any = true` -  Warn when a function returns 'Any' where a more specific type was expected.
    - `warn_unused_ignores = true` -  Warn if you have a '# type: ignore' comment that isn't actually silencing an error.
    - `warn_redundant_casts = true` -  Warn when a 'cast()' is used but the expression already has that type.
    - `strict_equality = true` -  Disallow comparing types that are not compatible (e.g., comparing int to None).
 - Controlling the 'Any` -  Type
    - `disallow_any_generics = true` -  Disallow usage of generic types (like List, Dict) without specific parameters.
    - `disallow_subclassing_any = true` -  Disallow inheriting from a class that is typed as 'Any'.
    - `disallow_any_explicit = true` -  Disallow explicit usage of the 'Any' type in your code.
    - `disallow_any_expr = false` -  Allow 'Any' in expressions; often kept false to avoid overwhelming errors in legacy code.
    - `disallow_any_decorated = true` -  Disallow 'Any' in functions that are decorated.
    - `disallow_any_unimported = true` -  Disallow 'Any' from types that aren't imported properly.
 - Configuration and Reporting
    - `warn_unused_configs = true` -  Warn if a section in your config file is never used (helps catch typos).
    - `show_error_codes = true` -  Show the specific code (e.g., [arg-type]) for every error to make filtering easier.

If strict mode is used, the following additional rules are also implemented
 - The `pytest-mypy` options are added to the `pytest` configuration.
 - The line length is restricted to 88 characters. This follows the `black` coding convention.
 - The following `ruff` linting rules are also put in place
    - `S` - `flake8-bandit`: check for common security issues
    - `N` - `pep8-naming`: check for PEP 8 naming conventions
    - `ERA` - `eradicate`: find and remove commented-out code
    - `B` - `flake8-bugbear`: find likely bugs and design problems
    - `C4` - `flake8-comprehensions`: improve list/dict/set comprehensions
    - `A` - `flake8-builtins`: check for shadowing of Python built-in symbols
    - `RET` - `flake8-return`: check for unnecessary return statements
    - `ISC` - `flake8-implicit-str-concat`: detect implicit string concatenation
    - `G` - `flake8-logging-format`: ensure proper logging format strings
    - `TRY` - `tryceratops`: prevent abuse of try/except blocks
    - `T` - `flake8-debugger`: catch tracepoints or debugger calls

Note that these rules are heavily restrictive, and it is left to the developer to decide which of these rules are relevant to the development model of the new application.


## Tools which can help you visualize your package

These tools aren't part of DUNE-DAQ but can be useful:

## Drawing with `draw.io`
Box diagrams help with digesting the contents of a repository, so the development of these diagrams has been integrated to make it as simple as possible. For `VSCode` or other graphical IDE users,
 - Download the `draw.io integration` or `draw.io for VSCode` extension.
 - Create a diagram in the `<repo_root>/diagrams/` directory
 - Open the file - this should automatically open the file in the `draw.io` interpreter

## Drawing with `mermaid`
Mermaid allows to draw diagrams with markdown. This includes UML, sequence, flowcharts, amongst others. An example is provided in `<daqpyutils_root>/diagrams/example.md`. This does not require compilation and allows for a diagram in the documentation to be updated without generating files, just be editing code. For `VSCode` or other graphical IDE users,
 - Download the `Mermaid` extension
 - When scripting your diagram, you can view the progress and validate correct syntax by previewing the diagram before committing.

## Of Historical Interest

Here are links to various talks and documents where the Python development standards were sorted out.

[Initial plans](https://docs.google.com/document/d/1i1CPLn7UYLbrUMF2IiQNtejUSnskyRpIolfuG46PweY/edit?tab=t.0#heading=h.8u53ddxujd8q)

[Slides 1](https://docs.google.com/presentation/d/1ttWGPtHcKsvJKeQOs3cHswES_xHRoXyYYxjmtbHYGT4/edit?slide=id.p#slide=id.p)

[Slides 2](https://docs.google.com/presentation/d/1Zl3YTXew__qPGyzRuToIhxjHpiGKYVDmpQsWqg0HmJk/edit?slide=id.g364767cc810_0_12#slide=id.g364767cc810_0_12)

[Slides 3](https://docs.google.com/presentation/d/1L9yofnW52rHQCHCo0Wo1ZjJhhWegAXlxs1FH4d5BG30/edit?slide=id.p#slide=id.p)

[daq-deliverables Issue proposing a standard layout](https://github.com/DUNE-DAQ/daq-deliverables/issues/191)

[daq-deliverables Issue discussing standardized tooling](https://github.com/DUNE-DAQ/daq-deliverables/issues/196)
