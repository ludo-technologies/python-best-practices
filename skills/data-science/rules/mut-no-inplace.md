---
title: Avoid inplace=True
impact: CRITICAL
impactDescription: No memory saving under Copy-on-Write; breaks chaining and hides data flow
tags: [pandas, inplace, immutability, copy-on-write]
---

# Avoid inplace=True [CRITICAL]

## Description
`inplace=True` promises to mutate a DataFrame without copying, but most implementations copy internally and rebind, so there is no performance win. It returns `None`, which breaks method chaining and causes `df = df.dropna(inplace=True)` bugs. It also hides data flow: a function that mutates its argument in place is harder to test and reason about than one that returns a new frame. PDEP-8 deprecates the keyword for most methods. Always use the returning form and assign the result.

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

# Classic bug: inplace returns None
df = df.fillna(0, inplace=True)  # df is now None
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
    )


df = clean(pd.read_csv("users.csv"))
df = df.fillna(0)
```

## Notes
- Treat DataFrames as immutable values inside functions: take a frame, return a frame. This matches `coding-standards/design-pure-functions`.
- The returning form composes with `pipe`, `assign`, and boolean indexing (see `style-method-chaining`).
- Under Copy-on-Write the returned frame shares memory with the input until either is written to, so the "extra copy" concern is moot.
- Rebinding the same name (`df = df.dropna()`) is fine and idiomatic; the point is that the operation is visible in the data flow.

## References
- [PDEP-8: Inplace methods in pandas](https://pandas.pydata.org/pdeps/0008-inplace-methods-in-pandas.html)
- [pandas User Guide - Copy-on-Write](https://pandas.pydata.org/docs/user_guide/copy_on_write.html)
