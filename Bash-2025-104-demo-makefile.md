# Step 1 — Why Makefile? (motivation)

* **Story:** your day starts with the same shell dance—clean, install, test, lint, build, run. One skipped step, one typo… and you lose an hour.
* **Aim:** one Makefile → a **reproducible**, **incremental** pipeline.
* **Principles:** declarative (say *what*), explicit, safe by default.
* **Outcome:** six verbs you’ll actually use: `clean → install → test → lint → build → run`.

```sh
# Before (fragile and forgetful)
rm -rf __pycache__ build dist
python3.11 -m venv .venv
. .venv/bin/activate && pip install -U pip build ruff pytest
. .venv/bin/activate && pip install -e .
. .venv/bin/activate && pytest
. .venv/bin/activate && ruff check .
. .venv/bin/activate && python -m build
. .venv/bin/activate && python -m first_app.main
```

**Explainer:** Make turns this sequence into named targets with order, failure, and idempotence handled for you.

**Explanation:** This step introduces the motivation for using a Makefile. It contrasts manual shell commands, which are error-prone and repetitive, with the structured approach of Make. By defining targets like "clean" or "install," you create a reliable workflow that handles dependencies automatically, making your development process more efficient and less frustrating. This sets the foundation for the rest of the tutorial, showing why we're building this system.

---

# Step 2 — Set up the project structure (for a buildable package)

* For beginners, we'll create a basic Python package structure that's easy to build and distribute.
* This uses a standard "src-layout" to keep source code separate from build artifacts and tests.
* We'll add a minimal `pyproject.toml` for modern packaging (including dependencies), a `README.md` for documentation, a simple `main.py` as the app entrypoint, and a basic test file.
* Since we're using `pyproject.toml` for dependencies, we'll define optional dev dependencies there (e.g., for tools like ruff and pytest). No runtime dependencies yet, but this structure allows easy addition later.

**Directory tree to create:**

```
first_app
├── src
│   └── first_app
│       └── main.py
├── tests
│   └── test_main.py
├── pyproject.toml
├── README.md
└── Makefile  (added in later steps)
```

**Create it step-by-step:**

1. Make the directories:

```sh
mkdir -p first_app/src/first_app first_app/tests
cd first_app
```

2. Add `main.py` (the tiniest app—it prints a greeting and the active Python version):

```python
# src/first_app/main.py
import sys

print("hello from", sys.version)
```

3. Add a basic test file (to make `test` target meaningful):

```python
# tests/test_main.py
import sys

def test_python_version():
    assert sys.version_info >= (3, 8), "Requires Python 3.8+"
```

4. Add `pyproject.toml` (minimal config for building the package with PEP 517/518, plus dev dependencies):

```toml
# pyproject.toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "first_app"
version = "0.1.0"
description = "A basic Python package example"
readme = "README.md"
requires-python = ">=3.8"
dependencies = []  # Runtime dependencies go here (empty for now)

[project.optional-dependencies]
dev = ["build", "ruff", "pytest"]  # Dev tools for testing, linting, building
```

5. Add `README.md` (simple placeholder):

```markdown
# First App

This is a basic Python package that prints a greeting and Python version.

## Usage

After building and installing, run `python -m first_app.main`.
```

**What you’ll see later:** When running, `hello from 3.11.x (...)`; tests will pass if Python is 3.8+.

**Explanation:** This step establishes the project's file structure and essential files. The "src-layout" separates your code from other elements, promoting cleanliness. `pyproject.toml` is key here—it's the modern standard for Python project configuration, handling build settings, metadata, and dependencies (both runtime and dev). By including dev tools like "build", "ruff", and "pytest" in [project.optional-dependencies.dev], we centralize everything in one file, eliminating the need for a separate requirements.txt. This makes the project more maintainable and ready for packaging/distribution.

---

# Step 3 — The six verbs (mental model)

* **clean**: remove build/test junk
* **install**: ensure tools + deps are present (using `pyproject.toml` and a virtual environment)
* **test**: run tests (prefer `pytest`; fallback to `unittest`)
* **lint**: style checks with `ruff`
* **build**: PEP 517 build to `dist/`
* **run**: execute as a module (`python -m first_app.main`)

**Rule of thumb:** if you can say it, you should have a target for it.

**Explanation:** This step outlines the core actions (verbs) that the Makefile will implement. Each verb corresponds to a common development task, forming a mental model for the workflow. By thinking in terms of these targets, you can invoke specific parts of the pipeline easily (e.g., "make test"). This abstraction simplifies complex sequences into simple commands, with dependencies handled automatically by Make.

---

# Step 4 — Safe defaults (Makefile header)

* Base interpreter is **overrideable**: `BASE_PYTHON ?= python3.11` (used to create the venv)
* We'll use a virtual environment (`.venv`) for isolation, with `PYTHON := .venv/bin/python`
* Safer shell recipes: `-euo pipefail`
* Default goal = `all` (so `make` runs the full workflow)

```makefile
BASE_PYTHON ?= python3.11
PYTHON := .venv/bin/python
OUT ?= dist
PKG_DIR ?= .

.DEFAULT_GOAL := all
SHELL := bash
.SHELLFLAGS := -euo pipefail -c
.PHONY: all clean venv install test lint build run help
.SUFFIXES:
.DELETE_ON_ERROR:
```

**Why:** `?=` lets you do `make BASE_PYTHON=python3.12 all`; `-euo pipefail` makes failures loud.

**Add this to your `Makefile` file in the `first_app/` directory.**

**Explanation:** This step sets up the Makefile's foundational configurations, like variables and phony targets. It introduces the virtual environment concept by defining `BASE_PYTHON` (for creating the venv) and `PYTHON` (for running commands inside it). These defaults ensure consistency and safety—e.g., failures stop the process immediately. This header is placed at the top of the Makefile and prepares the ground for adding specific targets without repeating configurations.

---

# Step 5 — Target anatomy (tab matters!)

* A target has **name : prerequisites** and a **tab-indented recipe**.
* Tabs (not spaces) are required in recipes.
* The **first** target is the default (we set `.DEFAULT_GOAL := all` instead).

```makefile
hello:
	@echo "Makefile skeleton OK"   # ← this line must start with a TAB
```

**Check yourself:** Append this to your `Makefile`, then run `make hello` — see the message? good.

**Explanation:** This step teaches the basic syntax of Make targets. A target defines a goal (like "hello"), optional prerequisites (dependencies), and recipes (commands, indented with tabs). Understanding this anatomy is crucial because all subsequent steps build on it—targets chain together via prerequisites, allowing Make to execute only what's necessary. The tab requirement is a common gotcha, so highlighting it prevents beginner errors.

---

# Step 6 — venv (create and set up virtual environment)

* Create a `.venv` directory if it doesn't exist, using the base Python.
* Upgrade pip inside the venv for reliability.
* This isolates dependencies and tools, preventing conflicts with system Python.

```makefile
venv: ## Create virtual environment
	@if [ ! -d .venv ]; then \
		echo "[venv] creating .venv with $(BASE_PYTHON)"; \
		$(BASE_PYTHON) -m venv .venv; \
	fi
	@. .venv/bin/activate && pip install -U pip
	@echo "[venv] ready"
```

**Why first (after header):** The venv ensures all tools and deps are isolated, making the project portable.

**Add this to your `Makefile` after the header.**

**Explanation:** This step adds a target to create a virtual environment (`.venv`), which is a self-contained Python setup. It checks if `.venv` exists to avoid recreation, uses `BASE_PYTHON` to initialize it, and upgrades pip. Virtual environments are best practice for Python projects because they isolate dependencies, avoiding version conflicts across projects. This target will be a prerequisite for others, ensuring everything runs in a controlled environment.

---

# Step 7 — clean (start from known-good state)

* Avoid stale artifacts; don’t delete source code or the venv.
* Keep it **targeted**: caches, eggs, wheels, build dirs.

```makefile
clean: ## Remove artifacts
	@rm -rf __pycache__ .pytest_cache .mypy_cache build dist *.egg-info
	@find . -name '*.pyc' -delete
	@echo "[clean] removed build/test artifacts"
```

**Why first in workflow:** clean reduces mysterious failures later.

**Add this to your `Makefile` after the venv target.**

**Explanation:** This step defines the "clean" target to remove temporary files generated during development, like caches and build outputs. It ensures a fresh start without affecting source code or the venv. Starting workflows with clean is important for reproducibility—stale files can cause inconsistent behavior. Note that we explicitly avoid removing `.venv` here; a separate target could be added later for that if needed.

---

# Step 8 — install (tools + project deps in venv)

* Depends on venv; installs dev tools and project deps from `pyproject.toml`.
* Uses editable install (`-e`) for development, allowing code changes without reinstall.
* No separate `requirements.txt`—everything is in `pyproject.toml`.

```makefile
install: venv ## Install tools + deps
	@. .venv/bin/activate && pip install -e ".[dev]"
	@echo "[install] done (python=$$( $(PYTHON) -V ))"
```

**Pattern:** Uses the venv for isolation; installs everything in one go via extras.

**Add this to your `Makefile`.**

**Explanation:** This step creates the "install" target, which depends on "venv" and handles dependency installation. It activates the venv and uses `pip install -e ".[dev]"` to install the project editable (for live code updates) along with dev tools specified in `pyproject.toml`. This approach centralizes dependencies, making the project easier to manage. If you add runtime deps later, they'd go in [project.dependencies], and this target would handle them automatically.

---

# Step 9 — test (works on minimal machines)

* Depends implicitly on install (via workflow); uses `pytest` (installed in dev deps).
* Quiet mode keeps the console readable.
* It will discover `tests/test_main.py`.

```makefile
test: ## Run tests (pytest or unittest)
	@$(PYTHON) -m pytest -q 2>/dev/null || $(PYTHON) -m unittest -q discover tests
	@echo "[test] ok"
```

**Teaching trick:** With our `test_main.py`, watch it run and pass.

**Add this to your `Makefile`.**

**Explanation:** This step adds the "test" target to execute unit tests. It prefers `pytest` (from dev deps) but falls back to Python's built-in `unittest` for robustness. The `-q` flag minimizes output for clarity. Tests are crucial for verifying code correctness; this target integrates them into the workflow, ensuring they run in the venv for consistency. The discovery mechanism automatically finds test files in `tests/`.

---

# Step 10 — lint (quality gate that can fail build)

* Uses `ruff` (from dev deps) for speed and clarity.
* Make fails if `ruff` fails.
* It will check `src/` and `tests/`.

```makefile
lint: ## Static checks (ruff)
	@$(PYTHON) -m ruff check . || (echo '[lint] ruff failed' >&2; exit 1)
	@echo "[lint] ok"
```

**Why fail fast:** style issues are cheapest to fix early.

**Add this to your `Makefile`.**

**Explanation:** This step introduces the "lint" target for code style and quality checks using `ruff`. It runs in the venv and explicitly fails the build on errors, enforcing standards early. Linting improves code readability and catches potential issues; integrating it here makes it a standard part of development. You can later configure `ruff` further in `pyproject.toml` under [tool.ruff] for custom rules.

---

# Step 11 — build (PEP 517, tool-agnostic)

* Modern builds via `python -m build` (uses our `pyproject.toml`).
* Output into `dist/`.
* Uses the venv's Python for consistency.

```makefile
build: ## Build wheel/sdist (PEP 517)
	@$(PYTHON) -m build -o $(OUT) $(PKG_DIR)
	@echo "[build] artifacts in ./$(OUT)"
```

**Note:** No assumptions about `setup.py` — future-proof. Our `pyproject.toml` makes this work.

**Add this to your `Makefile`.**

**Explanation:** This step defines the "build" target to create distributable packages (wheel and sdist) using Python's build module, guided by `pyproject.toml`. It runs in the venv to ensure the correct environment. Building packages is essential for sharing or deploying code; this target standardizes the process, producing artifacts in `dist/` for easy distribution via PyPI or otherwise.

---

# Step 12 — run (see it work)

* Execute your app as a module (thanks to editable install).
* Uses the venv's Python.

```makefile
run: ## Run the app
	@$(PYTHON) -m first_app.main
	@echo "[run] complete"
```

**Feedback loop:** you’ll see the active Python version in the output.

**Add this to your `Makefile`.**

**Explanation:** This step adds the "run" target to execute the application. It uses `python -m` to run it as an installed module, leveraging the editable install from earlier. This provides immediate feedback on code changes and runs in the isolated venv. It's the final verb in the workflow, allowing quick iteration without manual setup.

---

# Step 13 — Orchestrate with all (one command)

* Chain the verbs in the right order: clean first, then install (which handles venv), then the rest.
* Re-running `make` is safe; nothing dangerous happens if nothing changed.

```makefile
all: clean install test lint build run
```

**Read it aloud:** clean → install → test → lint → build → run.

**Add this right after the header in your `Makefile`.**

**Explanation:** This step creates the "all" target, which orchestrates the entire pipeline by listing prerequisites in sequence. Make handles execution order and skips unchanged parts for efficiency. This provides a single command (`make`) for the full workflow, embodying the reproducible pipeline goal. Note that "install" implicitly brings in "venv," ensuring isolation throughout.

---

# Step 14 — Capstone: complete Makefile (copy-paste)

```makefile
# ---- Makefile (foundation: clean, install, test, lint, build, run) ----
BASE_PYTHON ?= python3.11
PYTHON := .venv/bin/python
OUT ?= dist
PKG_DIR ?= .

.DEFAULT_GOAL := all
SHELL := bash
.SHELLFLAGS := -euo pipefail -c
.PHONY: all clean venv install test lint build run help
.SUFFIXES:
.DELETE_ON_ERROR:

all: clean install test lint build run

venv: ## Create virtual environment
	@if [ ! -d .venv ]; then \
		echo "[venv] creating .venv with $(BASE_PYTHON)"; \
		$(BASE_PYTHON) -m venv .venv; \
	fi
	@. .venv/bin/activate && pip install -U pip
	@echo "[venv] ready"

clean: ## Remove artifacts
	@rm -rf __pycache__ .pytest_cache .mypy_cache build dist *.egg-info
	@find . -name '*.pyc' -delete
	@echo "[clean] removed build/test artifacts"

install: venv ## Install tools + deps
	@. .venv/bin/activate && pip install -e ".[dev]"
	@echo "[install] done (python=$$( $(PYTHON) -V ))"

test: ## Run tests (pytest or unittest)
	@$(PYTHON) -m pytest -q 2>/dev/null || $(PYTHON) -m unittest -q discover tests
	@echo "[test] ok"

lint: ## Static checks (ruff)
	@$(PYTHON) -m ruff check . || (echo '[lint] ruff failed' >&2; exit 1)
	@echo "[lint] ok"

build: ## Build wheel/sdist (PEP 517)
	@$(PYTHON) -m build -o $(OUT) $(PKG_DIR)
	@echo "[build] artifacts in ./$(OUT)"

run: ## Run the app
	@$(PYTHON) -m first_app.main
	@echo "[run] complete"

```

**Replace your building `Makefile` with this complete version if needed.**

**Explanation:** This step compiles all previous elements into a full, copy-pasteable Makefile. It includes the header, all targets, and the "all" orchestrator. This capstone allows beginners to see the complete picture while understanding how pieces fit together. You can now use this as a template for real projects, customizing as needed.

---

# Step 15 — Hands-on

1. Follow Step 2 to create the directory structure, `main.py`, `test_main.py`, `pyproject.toml`, and `README.md`.

2. Save the complete Makefile from Step 14 as `Makefile` in `first_app/`.

3. Run:

```sh
make            # full pipeline (creates .venv if needed)
# or:
make venv
make clean
make install
make test
make lint
make build
make run
```

**Expect:** “hello from 3.11.x …” plus a `dist/` directory with wheel and sdist artifacts. Tests and lint should pass. A `.venv` folder will appear.

**Explanation:** This step provides practical exercises to apply the tutorial. By running commands, you verify the setup and see the workflow in action. It reinforces learning through hands-on experience, showing expected outputs and how the venv integrates. If issues arise, it highlights troubleshooting opportunities.

---

# Step 16 — Everyday use, overrides, next steps

**Everyday:**

* Quick checks: `make lint test`
* Rebuild only: `make build`
* Run again: `make run`

**Change interpreter:**

```sh
make BASE_PYTHON=python3.12 all  # Recreates .venv if needed
```

**Troubleshooting:**

* “Module not found”? Run `make install`.
* Pip permission issues? Use a user-level base Python or ensure permissions.
* Build fails? Check `pyproject.toml` syntax.
* Want to reset venv? Add a `clean-venv: rm -rf .venv` target, then run `make clean-venv install`.

**Next steps:**

* Add real tests to `tests/`.
* Add runtime dependencies to `[project.dependencies]` in `pyproject.toml` and rerun `make install`.
* Configure `ruff` in `pyproject.toml` under `[tool.ruff]`.
* Add a `help` target (self-documenting).
* Wire CI to run: `make clean install test lint build`.
* Add more fields to `pyproject.toml`, like `[tool.setuptools.packages.find]` for auto-package discovery.

**Explanation:** This final step guides ongoing usage and extension. It covers common commands, overrides (like changing Python version), and troubleshooting. Next steps suggest expansions, such as adding configs to `pyproject.toml` or integrating with CI, helping you evolve the project beyond basics.

---

### (Bonus) Optional `help` target you can drop in

```makefile
help:  ## Show target list
	@awk 'BEGIN{FS=":.*## "};/^[a-zA-Z0-9_.-]+:.*## /{printf "\033[36m%-14s\033[0m %s\n",$$1,$$2}' $(MAKEFILE_LIST)
```

Add `## short description` to any target to make it show up in `make help`.

**Explanation:** This bonus adds a self-documenting "help" target, listing all commented targets. It's useful for larger Makefiles, making the file easier to navigate without reading the whole thing.