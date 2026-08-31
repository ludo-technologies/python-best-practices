---
title: Validate DataFrames at Boundaries with pandera
impact: HIGH
impactDescription: Catch schema drift and bad values at load time instead of as wrong numbers downstream
tags: [pandas, validation, pandera, schema, data-quality]
---

# Validate DataFrames at Boundaries with pandera [HIGH]

## Description
Tabular data from files, databases, and APIs is untrusted in the same way request payloads are. A DataFrame with a renamed column, a negative price, or duplicated keys usually does not crash; it produces plausible-looking wrong results. `coding-standards/validation-pydantic` covers scalar records; pandera is the equivalent for DataFrames: declare column names, dtypes, nullability, and value constraints once, validate where data enters, and pass typed frames inward.

## Bad Example
```python
import pandas as pd


def load_orders(path: str) -> pd.DataFrame:
    df = pd.read_csv(path)
    # Hope the columns are what we expect
    return df


def revenue(df: pd.DataFrame) -> float:
    # Negative quantities, NaN prices, duplicate order_ids all flow through
    return float((df["price"] * df["qty"]).sum())
```

## Good Example
```python
import pandas as pd
import pandera.pandas as pa
from pandera.typing import DataFrame, Series


class OrderSchema(pa.DataFrameModel):
    order_id: Series[int] = pa.Field(unique=True)
    customer_id: Series[str] = pa.Field(str_length={"min_value": 1})
    price: Series[float] = pa.Field(ge=0)
    qty: Series[int] = pa.Field(gt=0)
    status: Series[str] = pa.Field(isin=["pending", "paid", "cancelled"])
    created_at: Series[pa.DateTime] = pa.Field(nullable=False)

    class Config:
        strict = True   # no unexpected columns


def load_orders(path: str) -> DataFrame[OrderSchema]:
    df = pd.read_csv(path, parse_dates=["created_at"])
    return OrderSchema.validate(df)  # raises SchemaError with the offending rows


@pa.check_types
def revenue(df: DataFrame[OrderSchema]) -> float:
    return float((df["price"] * df["qty"]).sum())
```

## Notes
- Validate once at the boundary (load, API response, upstream pipeline stage), not inside every transformation. Use `DataFrame[Schema]` type hints to document which functions expect validated input.
- `strict=True` catches renamed or extra columns.
- Be careful with `coerce=True`: it casts *before* checking, so a `float` column with `1.9` passes a `Series[int]` schema as `1`. Leave coercion off for numeric columns (a wrong dtype is a real error) and enable it per column (`pa.Field(coerce=True)`) only where the cast is a lossless normalization, e.g. `str` to `category`.
- Use `lazy=True` (`Schema.validate(df, lazy=True)`) in pipelines to collect all failures in one `SchemaErrors` report instead of stopping at the first.
- Add cross-column rules with `@pa.dataframe_check` (e.g., `end_date >= start_date`) and custom column rules with `@pa.check("col")`.
- A schema is also documentation: reviewers and agents can read the expected shape without opening a sample file.
- pandera also supports polars, pyspark, and dask via `pandera.polars` and similar modules.

## References
- [pandera Documentation](https://pandera.readthedocs.io/en/stable/)
- [pandera - DataFrameModel](https://pandera.readthedocs.io/en/stable/dataframe_models.html)
