## To add **test coverage reporting** using `pytest-cov`, follow these steps:

---

## ✅ Step 1: Install `pytest-cov`

If you haven't already, install with:

```bash
pip install pytest pytest-cov
```

---

## ✅ Step 2: Run Tests with Coverage

Run this in the root of your project:

```bash
pytest --cov=your_module_name --cov-report=term-missing
```

For example, if your permission logic is in a file called `myauth.py`, you'd run:

```bash
pytest --cov=myauth --cov-report=term-missing
```

This will:

* Show which lines are covered and which are missed.
* Highlight any decorators or logic not hit by your tests.

---

## ✅ Optional: Generate HTML Coverage Report

To view detailed coverage in a browser:

```bash
pytest --cov=myauth --cov-report=html
```

Then open:

```bash
htmlcov/index.html
```

---

## ✅ Example Output (Terminal)

```
----------- coverage: platform linux, python 3.11 -----------
Name             Stmts   Miss  Cover   Missing
----------------------------------------------
myauth.py           45      2    96%   32-33
----------------------------------------------
TOTAL               45      2    96%
```

---

## ✅ Pro Tip: Add to `pytest.ini`

To avoid typing flags every time, add a `pytest.ini`:

```ini
# pytest.ini
[pytest]
addopts = --cov=myauth --cov-report=term-missing
```

Then just run:

```bash
pytest
```
