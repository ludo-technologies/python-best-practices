---
title: Compare Numeric Results with Tolerance-Aware Assertions
impact: MEDIUM
impactDescription: No flaky float comparisons; failures show the first mismatching element, not "False"
tags: [numpy, pandas, testing, pytest, assert_allclose]
---

# Compare Numeric Results with Tolerance-Aware Assertions [MEDIUM]

## Description
`assert result == expected` on arrays raises `ValueError: The truth value of an array is ambiguous`, and `(result == expected).all()` is both exact (fails on `0.1 + 0.2`) and uninformative (prints `False`). NumPy and pandas ship assertion helpers that compare with relative/absolute tolerance, handle NaN placement, check dtype and index, and print a diff of the first mismatches. Use them in every test that touches arrays or frames.

## Bad Example
```python
import numpy as np


def test_normalize() -> None:
    result = normalize(np.array([1.0, 2.0, 3.0]))
    assert (result == np.array([-1.2247, 0.0, 1.2247])).all()  # exact compare fails


def test_summary() -> None:
    result = summarize(df)
    assert result.equals(expected)  # True/False only, no diff, exact float compare
    assert list(result.columns) == ["a", "b"]
```

## Good Example
```python
import numpy as np
import pandas as pd
import pytest


def test_normalize() -> None:
    result = normalize(np.array([1.0, 2.0, 3.0]))
    np.testing.assert_allclose(result, [-1.2247, 0.0, 1.2247], rtol=1e-4)


def test_normalize_scalar_stats() -> None:
    result = normalize(np.array([1.0, 2.0, 3.0]))
    assert result.mean() == pytest.approx(0.0, abs=1e-12)
    assert result.std() == pytest.approx(1.0)


def test_summary() -> None:
    result = summarize(df)
    expected = pd.DataFrame(
        {"a": [1.0, 2.5], "b": [3, 4]},
        index=pd.Index(["x", "y"], name="key"),
    )
    pd.testing.assert_frame_equal(result, expected, check_like=True)  # order-insensitive


def test_ids_exact() -> None:
    np.testing.assert_array_equal(assign_ids(3), np.array([0, 1, 2]))  # ints: exact is right
```

## Notes
- `np.testing.assert_allclose(actual, desired, rtol=1e-7, atol=0)`: set `atol` when expected values can be exactly zero, since relative tolerance alone cannot pass there.
- `assert_array_equal` is for integer, boolean, and string arrays; both helpers treat NaN in the same position as equal.
- `pd.testing.assert_frame_equal` / `assert_series_equal` check index, dtype, and column order by default; loosen deliberately with `check_dtype=False`, `check_like=True`, or `check_exact=False` and say why in the test.
- `pytest.approx` is right for scalars and simple sequences; it does not report which element differed or check dtype and shape.
- Build expected frames inline in the test (or with a factory fixture, see `testing/fixture-factory`) rather than loading golden CSVs; CSV round-trips lose dtypes and index names.

## References
- [NumPy - Test support (numpy.testing)](https://numpy.org/doc/stable/reference/routines.testing.html)
- [pandas API - Testing](https://pandas.pydata.org/docs/reference/testing.html)
