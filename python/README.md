# VS Code Python Setup

## Python Environment Setup (with .venv and uv)

1. **Create a virtual environment (.venv) in your project folder (Linux only):**

  ```bash
  python3 -m venv .venv
  source .venv/bin/activate
  ```

2. **Install [uv](https://github.com/astral-sh/uv) (a fast Python package manager):**

  ```bash
  pip install uv
  ```

3. **Use uv to install development tools:**

  ```bash
  uv pip install ruff mypy pre-commit
  ```

This will install [ruff](https://docs.astral.sh/ruff/), [mypy](http://mypy-lang.org/), and [pre-commit](https://pre-commit.com/) into your virtual environment for linting, type checking, and git hook management.

## Recommended Extensions
1. Ruff (Charliermarsh)
2. Mypy Type Checker (Microsoft)
3. Mintlify Doc Writer

## VS Code Settings
Add the following to your `.vscode/settings.json`:

```json
{
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.fixAll": "explicit",
      "source.organizeImports": "explicit"
    }
  },
  "mypy-type-checker.args": [
    "--config-file=${workspaceFolder}/pyproject.toml"
  ],
  "ruff.configuration": "${workspaceFolder}/pyproject.toml"
}
```

# Pre-commit & Docstring Setup

## 1. Install pre-commit hooks for Ruff and Mypy

1. Create a `.pre-commit-config.yaml` file in your project root with the following content:

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.11 # Or the latest version
    hooks:
      - id: ruff
        args: [ --fix ]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0 # Or the latest version
    hooks:
      - id: mypy
        additional_dependencies: [] # Add any other required libraries here
```

2. Install pre-commit and set up the git hook (run once):

```bash
pip install pre-commit
pre-commit install
```

---

## 2. How to Automatically Generate Docstrings (Recommended)

### Method 1: Use an AI-Powered IDE (Best & Most Seamless)
- **Cursor**: Highlight the function → Press Cmd+K (or Ctrl+K) → Type "Add Google-style docstring"
- **Copilot**: Highlight the function header → Right-click → Copilot → Generate Docs

Example output:
```python
def calculate_price(price: float, tax_rate: float) -> float:
    """
    Calculates the final price including tax.

    Args:
        price (float): The base price of the item.
        tax_rate (float): The tax rate (e.g., 0.07 for 7%).

    Returns:
        float: The total price rounded to 2 decimal places.
    """
    return round(price * (1 + tax_rate), 2)
```

### Method 2: Use the Free "Mintlify Doc Writer" Extension (Recommended!)
- Install the Mintlify Doc Writer extension in VS Code
- Go to the function line you want to document
- Press Ctrl + . (or highlight and right-click) and select "Generate Config"

### Method 3: Enforce Docstrings with Ruff (Self-discipline + AI Assistance)

Add the D (Pydocstyle) rule to your `pyproject.toml`:

```toml
[tool.ruff.lint]
# Add "D" to the select list
select = [
    # ... your existing rules ...
    "D", # Pydocstyle: Enforce docstring requirements
]

[tool.ruff.lint.pydocstyle]
convention = "google" # Or "numpy" as preferred
```

---

Follow these steps to ensure your Python code remains high-quality and easy to maintain in the long term!
