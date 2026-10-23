*This project has been created as part of the 42 curriculum by gvan-box.*

# Python Modules

## Description

The Python modules are a series of short, themed exercise sets that teach Python from the basics up to object-oriented design, packaging, data validation and functional programming. Each module is a folder of exercises (`ex0`, `ex1`, ...) that are run and checked separately.

| Module | Topic | What it covers |
|---|---|---|
| 00 | Getting started | Printing, formatting time, inspecting object types, `None`/falsy values |
| 02 | Exceptions | Raising and catching exceptions, handling different error types (`ValueError`, `ZeroDivisionError`, `FileNotFoundError`, `KeyError`, ...) |
| 03 | Command-line arguments | Reading and validating `sys.argv` |
| 06 | Imports and packages | Packages with `__init__.py`, absolute and relative imports, aliases, and breaking circular imports with late imports and dependency injection |
| 07 | Object-oriented design | A trading-card game: abstract base classes, inheritance, multiple inheritance with interfaces (`Combatable`, `Magical`, `Rankable`), the Factory and Strategy patterns, and a tournament platform |
| 08 | Environments and dependencies | Detecting a virtual environment, managing dependencies with `pip`/`requirements.txt` or Poetry (pandas, NumPy, matplotlib), and configuration through environment variables and `.env` files |
| 09 | Data validation | Pydantic models with `Field` constraints, enums, custom `model_validator`s and nested models (space station, alien contact, crew mission) |
| 10 | Functional programming | Lambdas, higher-order functions, closures and `nonlocal`, `functools` (`reduce`, `partial`, `lru_cache`, `singledispatch`) and decorators with `wraps` |

## Instructions

Each exercise is a standalone script, run from its module folder:

```bash
python3 ex0/lambda_spells.py
```

Module 06 is run from the module root so the `alchemy` package can be imported:

```bash
cd python06
python3 ft_sacred_scroll.py
```

Modules 08 and 09 need third-party packages. Install them in a virtual environment first:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install pydantic pandas numpy matplotlib python-dotenv
```

## Highlights

**Module 07: card game architecture.** Cards are built from an abstract `Card` base class. Elite cards combine several interfaces through multiple inheritance, a `CardFactory` creates themed cards, and the `GameEngine` takes a pluggable `GameStrategy`, so the play style can change without touching the engine.

**Module 09: validation at the boundary.** Every input goes through a Pydantic model, so invalid data fails early with a clear `ValidationError` instead of spreading through the program. Cross-field rules (for example, mission requirements that depend on the crew's ranks) are written as `model_validator`s.

**Module 10: functions as building blocks.** Closures keep private state without a class, `lru_cache` memoises expensive calls, `singledispatch` picks an implementation by argument type, and decorators add retry, validation and timing behaviour without changing the wrapped function.

## Resources

- [The Python Tutorial](https://docs.python.org/3/tutorial/)
- [Python docs: modules and packages](https://docs.python.org/3/tutorial/modules.html)
- [Python docs: `abc`](https://docs.python.org/3/library/abc.html) and [`functools`](https://docs.python.org/3/library/functools.html)
- [Pydantic documentation](https://docs.pydantic.dev/)
- [PEP 8](https://peps.python.org/pep-0008/) and [PEP 484](https://peps.python.org/pep-0484/)

### AI usage

This README was written with Claude (Anthropic), based on the project's source code.
