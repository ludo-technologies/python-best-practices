---
title: Use category Dtype for Low-Cardinality Strings
impact: HIGH
impactDescription: 5-20x less memory and faster groupby/sort/compare on repeated labels
tags: [pandas, dtype, category, memory, groupby]
---

# Use category Dtype for Low-Cardinality Strings [HIGH]

## Description
A string column with a few distinct values repeated across millions of rows stores every occurrence separately. `category` dtype stores each distinct value once and an integer code per row, which cuts memory dramatically and speeds up `groupby`, `sort_values`, `merge`, and equality comparisons. It also lets you declare a domain (allowed values) and an explicit order for ordinal data such as sizes or severity levels.

## Bad Example
```python
import pandas as pd

df = pd.read_csv("events.csv")  # 20M rows
# event_type: str, 8 distinct values -> ~1.2 GB
# severity: str, "low"/"medium"/"high" -> sorts alphabetically: high, low, medium

summary = df.groupby("event_type")["duration"].mean()  # slow hashing of strings
df.sort_values("severity")  # wrong order
```

## Good Example
```python
import pandas as pd

SEVERITY = pd.CategoricalDtype(["low", "medium", "high"], ordered=True)

df = pd.read_csv(
    "events.csv",
    dtype={"event_type": "category", "severity": SEVERITY},
)
# event_type: ~20 MB; severity has a defined order and domain

summary = df.groupby("event_type", observed=True)["duration"].mean()
df.sort_values("severity")                       # low, medium, high
critical = df[df["severity"] >= "medium"]        # ordinal comparison works

# Converting an existing column
df["region"] = df["region"].astype("category")
```

## Notes
- Rule of thumb: convert when distinct values are well under ~50% of rows; high-cardinality columns (free text, IDs) gain nothing and may use more memory.
- Pass `observed=True` to `groupby` on categoricals to avoid emitting rows for unobserved categories (the default since pandas 3.0).
- Assigning a value outside the declared categories raises `TypeError`; add it first with `cat.add_categories`. This is a feature: unexpected labels fail fast.
- Operations that combine frames (`concat`, `merge`) keep `category` only if both sides share identical categories; otherwise the result silently falls back to a plain string dtype. Define a shared `CategoricalDtype` constant and apply it to every source.
- Access codes with `df["region"].cat.codes` when a compact integer representation is needed for a model input.

## References
- [pandas User Guide - Categorical data](https://pandas.pydata.org/docs/user_guide/categorical.html)
