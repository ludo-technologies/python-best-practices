---
title: Avoid inplace=True
impact: CRITICAL
impactDescription: Rarely saves memory; breaks chaining, returns None for most methods, and hides data flow
tags: [pandas, inplace, immutability, copy-on-write]
---

# Avoid inplace=True [CRITICAL]

## Description
`inplace=True` reads as "mutate without copying", but that is only true for a minority of methods. Shape-changing methods (`dropna`, `sort_values`, `rename`, `reset_index`, `drop`, ...) cannot modify the buffer in place: they build a new frame, rebind it under the hood, and return `None`, so `df = df.dropna(inplace=True)` leaves `df` as `None`. Value-changing methods (`fillna`, `replace`, `clip`, `where`, `mask`, `interpolate`) can update the buffer under Copy-on-Write and return `self` in pandas 3.0, but the keyword still breaks method chaining and hides the fact that the caller's data was modified. PDEP-8 plans to deprecate `inplace` for the first group. Default to the returning form and assign the result; treat `inplace=True` as a deliberate, commented memory optimization for a value-changing method on a very large frame.

## Bad Example
```python
import pandas as pd


def clean(df: pd.DataFrame) -> None:
    df.dropna(subset=["email"], inplace=True)
    df.rename(columns={"e-mail": "email"}, inplace=True)
    df.sort_values("created_at", inplace=True)
    df.reset_index(drop=True, inplace=True)


df = pd.read_csv("users.csv")
clean(df)  # caller cannot tell df was modified

# Classic bug: shape-changing methods return None with inplace=True
df = df.dropna(inplace=True)  # df is now None
```

## Good Example
```python
import pandas as pd


def clean(df: pd.DataFrame) -> pd.DataFrame:
    return (
        df.dropna(subset=["email"])
        .rename(columns={"e-mail": "email"})
        .sort_values("created_at")
        .reset_index(drop=True)
        .fillna({"plan": "free"})
    )


df = clean(pd.read_csv("users.csv"))
```

## Notes
- Treat DataFrames as immutable values inside functions: take a frame, return a frame. This matches `coding-standards/design-pure-functions`.
- The returning form composes with `pipe`, `assign`, and boolean indexing (see `style-method-chaining`).
- Under Copy-on-Write the returned frame shares memory with the input until one of them is written to, so for shape-changing methods there was never a copy to save.
- When a value-changing method on a frame that fills most of memory really must avoid a second buffer, `df.fillna(0, inplace=True)` is legitimate in pandas 3.0; keep it at the top level (not inside a helper that received `df` as an argument) and comment why.
- Rebinding the same name (`df = df.dropna()`) is fine and idiomatic; the point is that the operation is visible in the data flow.

## References
- [PDEP-8: Inplace methods in pandas](https://pandas.pydata.org/pdeps/0008-inplace-methods-in-pandas.html)
- [pandas User Guide - Copy-on-Write](https://pandas.pydata.org/docs/user_guide/copy_on_write.html)
