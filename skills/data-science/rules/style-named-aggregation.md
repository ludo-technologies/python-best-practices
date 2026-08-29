---
title: Use Named Aggregation in groupby
impact: MEDIUM
impactDescription: Flat, explicitly named result columns instead of MultiIndex surprises
tags: [pandas, groupby, agg, style]
---

# Use Named Aggregation in groupby [MEDIUM]

## Description
`groupby().agg({"col": ["sum", "mean"]})` returns a MultiIndex column header that then needs flattening, renaming, and guessing about order. `groupby().apply(custom_func)` runs a Python loop per group. Named aggregation (`agg(new_name=("column", func))`) declares the output column name, source column, and reducer in one place, produces a flat frame, and keeps the fast built-in reducers.

## Bad Example
```python
import pandas as pd

summary = df.groupby("region").agg({"revenue": ["sum", "mean"], "order_id": "count"})
# columns: MultiIndex [('revenue','sum'), ('revenue','mean'), ('order_id','count')]
summary.columns = ["_".join(c) for c in summary.columns]
summary = summary.rename(columns={"order_id_count": "orders"}).reset_index()


# Per-group Python function for something a built-in reducer covers
def stats(g: pd.DataFrame) -> pd.Series:
    return pd.Series({"total": g["revenue"].sum(), "orders": len(g)})


summary = df.groupby("region").apply(stats)
```

## Good Example
```python
import pandas as pd

summary = df.groupby("region", as_index=False).agg(
    total_revenue=("revenue", "sum"),
    avg_revenue=("revenue", "mean"),
    orders=("order_id", "count"),
    first_order=("created_at", "min"),
)
# columns: region, total_revenue, avg_revenue, orders, first_order

# Custom reducer still fits the same shape
summary = df.groupby("region", as_index=False).agg(
    p95_revenue=("revenue", lambda s: s.quantile(0.95)),
)
```

## Notes
- Prefer string names for built-in reducers (`"sum"`, `"mean"`, `"count"`, `"nunique"`, `"min"`, `"max"`, `"first"`); they dispatch to compiled implementations, while a lambda runs per group in Python.
- `as_index=False` keeps the group keys as columns, avoiding a trailing `reset_index()`.
- For per-row results aligned with the original frame (e.g., group mean broadcast back), use `groupby(...).transform("mean")` rather than merging the summary back in.
- `pd.NamedAgg(column=..., aggfunc=...)` is the explicit form of the tuple if readability calls for it.
- The same syntax works on `Series.groupby` and on `resample`/`rolling` objects.

## References
- [pandas User Guide - Named aggregation](https://pandas.pydata.org/docs/user_guide/groupby.html#named-aggregation)
