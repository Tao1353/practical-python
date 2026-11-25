# AI Review of fileparse.py and portfolio.py

|AI suggestion|Accepted?|Reason|
|:---:|:---:|:---:|
|Use `next(rows, [])` to read the header row|Yes|Prevents a `StopIteration` crash when the input file is empty|
|Wrap `headers.index()` in try/except and raise a clear `RuntimeError`|Yes|The default `ValueError` message is vague; the new message names the missing column|
|Catch `IndexError` along with `ValueError` for bad rows|Yes|Rows with too few columns crashed the parser; now they are skipped or reported like other bad rows|
|Move the `select` step inside the same try block as type conversion|Yes|Short rows can fail while selecting columns, so both steps need the same error handling|
|Start row numbering at 2 when `has_headers` is True|Yes|The first data row is line 2 of the file, so error messages now point to the correct line|
|Print errors to `sys.stderr` as a single message|Yes|Error output stays separate from normal output, and one message per bad row is easier to read|
|Fix line wrapping and spacing, and add `import sys`|Yes|Follows PEP 8 style and is needed for the `stderr` change|
|Expand the `parse_csv` docstring with Args and Returns|Yes|Documents the parameters, including the note that `zip()` stops at the shorter list|
|Use `iter(self._holdings)` in `__iter__`|Yes|Cleaner than calling the dunder method `__iter__` directly|
|Move `from collections import Counter` to the top of `portfolio.py`|Yes|PEP 8 puts imports at the top of the file|
|Rename `self = cls()` to `portfolio = cls()` in `from_csv`|Yes|Using `self` in a classmethod is confusing because it is really a new instance|
|Remove trailing whitespace and extra blank lines at the end of `portfolio.py`|Yes|Formatting cleanup that linters flag|
|Add docstrings to the class and its main methods|Yes|Makes the code easier to read and use|
|Replace `s.shares * s.price` with `s.cost` in `total_cost`|No|Only works if `Stock` has a `cost` property, and I would need to check my `stock.py` first|
|Guard against `select` or `types` being passed in `**opts` of `from_csv`|No|Not needed for this assignment, since `from_csv` is only called with other options|
|Run `flake8` on both files|No|A tool to use in the future, not a change to the code itself|
