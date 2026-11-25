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

# Edge-Case Report: fileparse.py and portfolio.py

I ran each of the 15 data rows in `edge_cases.csv` through the parser one at a time (with the header), then as a whole file, using the same options `Portfolio.from_csv` uses:

```python
select=['name', 'shares', 'price']
types=[str, int, float]
```

## Results by row

|Line|Row|Before|After|Decision and reason|
|:---:|:---|:---|:---|:---|
|2|`AA,100,`|Skipped (reported as "Row 1")|Skipped, "Row 2"|**Should fail.** Price is missing and `0.0` would hide missing data|
|3|`IBM,,91.10`|Skipped|Skipped|**Should fail.** Shares are missing|
|4|`"Smith, Jones & Co",50,12.5`|Parsed correctly|Parsed correctly|**Correct.** The `csv` module handles the quoted comma|
|5|`GE,abc,23.45`|Skipped|Skipped|**Should fail.** `abc` is not a number|
|6|`MSFT,10.5,45.00`|Skipped|Skipped|**Should fail by design.** `types` says `int`; fractional shares would need `float`|
|7|`CAT,-100,83.44`|**Accepted (wrong)**|Skipped: negative shares|**Fixed** in `Portfolio`. Negative shares make `total_cost` negative; short positions need their own design|
|8|`HPQ,50`|**Raised `IndexError`**, which aborted the whole file|Skipped: expected 3 fields, got 2|**Fixed** by the field-count check|
|9|`,200,15.00`|**Accepted (wrong)**, blank name|Skipped: missing name|**Fixed** in `Portfolio`. A blank name cannot be looked up and would group incorrectly in `tabulate_shares`|
|10|`KO,1,000,45.10`|**Accepted (wrong)** as KO, 1 share, price 0.0. The real price was silently dropped|Skipped: expected 3 fields, got 4|**Fixed** by the field-count check. This is the most dangerous case because an unquoted comma silently changes the data|
|11|`PG,100,$31.20`|Skipped|Skipped|**Should fail.** Stripping `$` is a data-cleaning choice, and the currency is unspecified, so I left it strict|
|12|`  AMZN  , 20 ,1500.00`|**Accepted (wrong)** with the name `'  AMZN  '`, so `'AMZN' in portfolio` was `False`|Accepted as `'AMZN'`|**Fixed** with `strip=True`|
|13|`T,1e3,22.5`|Skipped|Skipped|**Should fail.** `int('1e3')` is invalid; converting through `float` could hide rounding problems|
|14|`F,100,NaN`|**Accepted (wrong)**, `total_cost` becomes `nan`|Skipped: price must be finite|**Fixed** in `Portfolio` with `math.isfinite`|
|15|`XOM,100,45.10,extra,fields`|**Accepted (wrong)**, extra fields silently ignored|Skipped: expected 3 fields, got 5|**Fixed** by the field-count check. It has the same shape as row 10, so it cannot be trusted|
|16|`"BA,100,98.25`|**Raised `IndexError`**|Skipped: expected 3 fields, got 1|**Should fail.** The quote is never closed and `csv` cannot recover|

**Totals for the original parser:** 2 rows raised, 6 were silently accepted with wrong or suspect data (rows 7, 9, 10, 12, 14, and 15), 6 were correctly rejected, and 1 was correct. Because the first raised row aborted the whole-file test, the original parser could not complete it.

**After the fixes:** `Portfolio.from_csv` on the whole file keeps 2 holdings (Smith, Jones & Co and AMZN, `total_cost` 30625.0) and reports the other 13 rows on stderr with a reason.

## What changed

**fileparse.py**

- Field count: with headers, a row whose field count differs from the header's is rejected. This catches rows 8, 10, 15, and 16.
- Whitespace: the new `strip=True` option removes leading and trailing whitespace from headers and values.
- Line numbers: row numbers now use `rows.line_num`, so they remain correct when a quoted field spans lines.

**portfolio.py**

- New `_check_holding()` rejects a blank name, negative shares, and a negative or non-finite price.
- It respects `silence_errors`.

## Other cases checked

|Case|Result|
|:---|:---|
|`inf`, `-inf`, negative price|Rejected|
|Zero shares|Accepted on purpose: harmless and not clearly invalid|
|Header-only file|Empty portfolio|
|Columns in a different order|Parsed correctly|
|Empty file or missing `price` column|`RuntimeError` naming the column because no data can be parsed|

## Known limitation

An unterminated quote in the middle of a file swallows the lines after it into one field. In my test, `BBB`, `CCC`, and `DDD` all merged into one bad row, so those three rows are lost and reported only through a single error message.

## Bad rows the AI found that I had not considered

My original code handled bad numbers and blank lines, so those were probably already on my radar. The cases below were ones I had not accounted for.

1. **Unquoted comma inside a number (`KO,1,000,45.10`)**: no error at all, only wrong data. This is the one most worth remembering.
2. **Extra fields (`XOM`)**: the same root cause and it only worked by accident.
3. **Whitespace padding**: values converted successfully but broke lookups.
4. **`NaN` and `inf`**: `float()` accepts them, and one bad price poisons `total_cost`.
5. **Valid type, invalid meaning**: negative share counts or blank names convert without error, so type conversion alone does not validate data.
6. **Short rows and unterminated quotes**: these raised uncaught `IndexError` exceptions, so one bad row could lose the whole file.
7. **Formats that should fail**: `$` prefixes, scientific notation (`1e3`), and decimal shares are rejected correctly, but they are real-world patterns.
8. **Unterminated quote mid-file**: it was generated as the last row initially, so I also tested the mid-file version; it swallowed the following rows.

# Bug Implementation

The injected bug was in `main()`: the two lines

```python
filename = args[1]
print('Total cost:', portfolio_cost(filename))
```

were indented one level too far, placing them inside

```python
if len(args) != 2:
```

## Why it was hard to find

- **No error:** the file is valid Python, so it produces no `IndentationError` or traceback.
- **Silent failure:** the program does not crash. It simply does nothing.
- **Natural appearance:** the lines look consistently indented under the `if`, so the block reads plausibly at a glance.
- **No obvious clue:** the lines come after a `raise`, making the issue easy to miss.

My partner found the error in 10 minutes.
