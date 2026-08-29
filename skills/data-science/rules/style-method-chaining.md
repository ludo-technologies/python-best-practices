---
title: Express Transformations as Method Chains
impact: MEDIUM
impactDescription: Readable pipelines with no half-transformed intermediate variables to mismanage
tags: [pandas, style, assign, pipe, method-chaining]
---

# Express Transformations as Method Chains [MEDIUM]

## Description
A sequence of column assignments and filters on a mutable `df` interleaves reads and writes, forces the reader to track which columns exist at each line, and invites reuse of a half-transformed frame. Chaining `assign`, boolean indexing, `rename`, `pipe`, and friends makes a transformation a single expression: inputs at the top, output at the bottom, every step visible. `pipe` lets you insert your own functions without breaking the chain.

## Bad Example
```python
import pandas as pd

df = pd.read_csv("sales.csv")
df["revenue"] = df["price"] * df["qty"]
df = df[df["revenue"] > 0]
df["month"] = df["date"].dt.to_period("M")
df2 = df.groupby("month")["revenue"].sum()
df2 = df2.reset_index()
df2.columns = ["month", "total_revenue"]
df2 = df2.sort_values("total_revenue", ascending=False)
```

## Good Example
```python
import pandas as pd


def add_derived_columns(df: pd.DataFrame) -> pd.DataFrame:
    return df.assign(
        revenue=lambda d: d["price"] * d["qty"],
        month=lambda d: d["date"].dt.to_period("M"),
    )


monthly = (
    pd.read_csv("sales.csv", parse_dates=["date"])
    .pipe(add_derived_columns)
    .loc[lambda d: d["revenue"] > 0]
    .groupby("month", as_index=False)
    .agg(total_revenue=("revenue", "sum"))
    .sort_values("total_revenue", ascending=False)
)
```

## Notes
- Use `lambda d: ...` inside `assign` and `.loc` when a step depends on a column created earlier in the same chain; the lambda receives the intermediate frame.
- Wrap the chain in parentheses and put one method per line; this is the layout `ruff format` preserves.
- Break long chains into named functions joined with `pipe` rather than into intermediate variables. Each function then takes a frame and returns a frame, which is easy to unit test.
- Chains are not free: each step returns a new object (lazily copied under Copy-on-Write). For hot loops on small frames a direct operation can be appropriate; measure before deviating.
- Prefer boolean indexing with a lambda over `query`/`eval` strings so column references stay as Python code that linters and type checkers can see.

## References
- [pandas API - DataFrame.assign](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.assign.html)
- [pandas API - DataFrame.pipe](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.pipe.html)
