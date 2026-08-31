---
title: Vectorize Instead of Python Loops
impact: CRITICAL
impactDescription: 10-1000x faster than iterrows, row-wise apply, or element-wise loops
tags: [vectorization, numpy, pandas, performance, iterrows, apply]
---

# Vectorize Instead of Python Loops [CRITICAL]

## Description
NumPy and pandas execute operations in compiled code over whole arrays. A Python `for` loop, `DataFrame.iterrows()`, or `apply(func, axis=1)` drops back to the interpreter for every element, paying Python overhead and boxing/unboxing per value. Express computations as whole-array operations: arithmetic, comparisons, boolean masks, `np.where`/`np.select`, and pandas string/datetime accessors.

Note that `coding-standards/perf-list-comprehension` does not apply here: a comprehension over an array is still a Python loop.

## Bad Example
```python
import numpy as np
import pandas as pd


# Element-wise Python loop over an array
def normalize(values: np.ndarray) -> np.ndarray:
    result = []
    for v in values:
        result.append((v - values.mean()) / values.std())
    return np.array(result)


# Row-wise iteration over a DataFrame
def add_discount(df: pd.DataFrame) -> pd.DataFrame:
    discounts = []
    for _, row in df.iterrows():
        if row["quantity"] >= 100:
            discounts.append(row["price"] * 0.2)
        elif row["quantity"] >= 10:
            discounts.append(row["price"] * 0.1)
        else:
            discounts.append(0.0)
    df["discount"] = discounts
    return df


# apply with axis=1 is still a Python loop
df["total"] = df.apply(lambda row: row["price"] * row["quantity"], axis=1)
```

## Good Example
```python
import numpy as np
import pandas as pd


def normalize(values: np.ndarray) -> np.ndarray:
    return (values - values.mean()) / values.std()


def add_discount(df: pd.DataFrame) -> pd.DataFrame:
    conditions = [df["quantity"] >= 100, df["quantity"] >= 10]
    rates = [0.2, 0.1]
    rate = np.select(conditions, rates, default=0.0)
    return df.assign(discount=df["price"] * rate)


df["total"] = df["price"] * df["quantity"]

# String and datetime operations via accessors, not apply
df["domain"] = df["email"].str.split("@").str[1]
df["year"] = df["signup_at"].dt.year

# Boolean masks instead of filtering in a loop
active_adults = df[(df["age"] >= 18) & (df["status"] == "active")]
```

## Notes
- Escalation order when a loop seems unavoidable: arithmetic/masks → `np.where` / `np.select` → `Series.map` with a dict → `Series.apply` (element-wise) → `numba` / `Cython`. Reach for `iterrows` only for side effects on a handful of rows.
- `DataFrame.apply(..., axis=0)` on columns is fine when the function is itself vectorized; it is `axis=1` (row-wise) that loops in Python.
- `itertuples()` is ~10x faster than `iterrows()` if you truly must iterate, but vectorizing is faster still.
- Group-wise logic: use `groupby(...).transform(...)` / `.agg(...)` with built-in reducers (`"sum"`, `"mean"`) rather than `groupby().apply(python_func)`.
- Vectorized code also reads better: it states *what* is computed instead of *how* to iterate.

## References
- [pandas User Guide - Enhancing performance](https://pandas.pydata.org/docs/user_guide/enhancingperf.html)
- [NumPy - Broadcasting](https://numpy.org/doc/stable/user/basics.broadcasting.html)
