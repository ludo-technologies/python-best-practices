---
title: Assign with .loc, Never with Chained Indexing
impact: CRITICAL
impactDescription: Chained assignment silently does nothing under Copy-on-Write (pandas 3.0 default)
tags: [pandas, indexing, loc, copy-on-write, chained-assignment]
---

# Assign with .loc, Never with Chained Indexing [CRITICAL]

## Description
`df[mask]["col"] = value` and `df["col"][mask] = value` are two operations: the first creates a new object, the second mutates that temporary. With Copy-on-Write (opt-in since pandas 2.0, the only behavior in pandas 3.0) the original `df` is never modified, and pandas emits a `ChainedAssignmentError` warning. Before CoW, the outcome depended on internal view/copy details and produced the infamous `SettingWithCopyWarning`. Use a single `.loc[row_indexer, col_indexer] = value` so the write targets `df` itself.

## Bad Example
```python
import pandas as pd

df = pd.read_csv("orders.csv")

# Chained assignment: writes to a temporary, df is unchanged
df[df["status"] == "pending"]["priority"] = "high"
df["priority"][df["amount"] > 1000] = "urgent"

# Same problem via a filtered intermediate
pending = df[df["status"] == "pending"]
pending["priority"] = "high"  # does not touch df; may not even be intended
```

## Good Example
```python
import pandas as pd

df = pd.read_csv("orders.csv")

# Single indexing operation targets df directly
df.loc[df["status"] == "pending", "priority"] = "high"
df.loc[df["amount"] > 1000, "priority"] = "urgent"

# Multiple columns at once
df.loc[df["status"] == "cancelled", ["priority", "assignee"]] = ["none", None]

# If you want an independent subset, say so explicitly
pending = df.loc[df["status"] == "pending"].copy()
pending["priority"] = "high"  # clearly a separate object
```

## Notes
- Under CoW, every indexing operation that returns a DataFrame/Series behaves like a copy; mutating a derived object never propagates to the parent. Design code around that rule instead of relying on views.
- `df["col"] = ...` (single bracket, whole column) is fine: it is one `__setitem__` on `df`.
- Use `.loc` for label-based and `.iloc` for position-based selection; do not mix positional integers into `.loc` on a non-integer index.
- Assigning many derived columns is often clearer with `assign` (see `style-method-chaining`).
- `pd.options.mode.copy_on_write = True` enables CoW in pandas 2.x to surface these bugs before upgrading to 3.0.

## References
- [pandas User Guide - Copy-on-Write](https://pandas.pydata.org/docs/user_guide/copy_on_write.html)
- [pandas User Guide - Indexing and selecting data](https://pandas.pydata.org/docs/user_guide/indexing.html)
