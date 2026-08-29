---
title: Specify Dtypes When Loading Data
impact: HIGH
impactDescription: Prevents silent type inference errors and cuts memory use, often 2-10x
tags: [pandas, dtype, read_csv, io, memory]
---

# Specify Dtypes When Loading Data [HIGH]

## Description
`read_csv` and friends infer dtypes from the data they see. Inference is fragile: an ID column with leading zeros becomes `int64` and loses them, a column with one blank cell becomes `float64`, dates stay as `object` strings, and large files are read in chunks so dtypes can differ between runs. Declare the schema at the boundary with `dtype=`, `parse_dates=`, and `usecols=`, and treat loading as the point where raw bytes become typed data.

## Bad Example
```python
import pandas as pd

df = pd.read_csv("transactions.csv")
# zip_code: "02134" -> 2134 (int64)
# customer_id: int64 until one row is missing, then float64 with NaN
# amount: object if a single row contains "N/A"
# created_at: object (string) - .dt accessor fails
# region: object with 5 distinct values repeated 10M times

df["created_at"] = pd.to_datetime(df["created_at"])  # fixed later, maybe
```

## Good Example
```python
import pandas as pd

TRANSACTION_DTYPES = {
    "customer_id": "Int64",       # nullable integer, keeps NaN without float cast
    "zip_code": "string",         # preserves leading zeros
    "amount": "float64",
    "region": "category",
}

df = pd.read_csv(
    "transactions.csv",
    usecols=[*TRANSACTION_DTYPES, "created_at"],
    dtype=TRANSACTION_DTYPES,
    parse_dates=["created_at"],
    na_values=["N/A", ""],
)

# Prefer typed formats over CSV when you control the producer
df.to_parquet("transactions.parquet")
df = pd.read_parquet("transactions.parquet")  # schema travels with the data
```

## Notes
- Use nullable dtypes (`"Int64"`, `"boolean"`, `"string"`) for columns that may contain missing values; NumPy `int64`/`bool` cannot represent NaN and will be upcast.
- `dtype_backend="pyarrow"` gives Arrow-backed columns with lower memory and faster string operations; pandas 3.0 already uses a dedicated `str` dtype by default instead of `object`.
- Downcast numerics when the range allows (`float32`, `int32`) for large datasets; measure with `df.memory_usage(deep=True)`.
- Validate the result after loading with a schema (see `schema-pandera`) so a schema drift in the source fails fast instead of corrupting downstream numbers.
- For NumPy, the same principle applies to constructors: `np.zeros(n, dtype=np.float32)` rather than relying on the `float64` default (see `dtype-numpy-explicit`).

## References
- [pandas User Guide - IO tools](https://pandas.pydata.org/docs/user_guide/io.html)
- [pandas User Guide - Nullable integer data type](https://pandas.pydata.org/docs/user_guide/integer_na.html)
