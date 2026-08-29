# Section Definitions

## Vectorization (vec)
**Impact:** CRITICAL

Keep computation in compiled NumPy/pandas code instead of the Python interpreter.
Covers replacing loops and row-wise apply with array operations, and building arrays and frames in one step.

**Rules:**
- `vec-no-python-loops` - Vectorize instead of Python loops, iterrows, and row-wise apply
- `vec-no-incremental-growth` - Build arrays and DataFrames once, not incrementally

## Indexing & Mutation (mut)
**Impact:** CRITICAL

Write to DataFrames in ways that are correct under Copy-on-Write and visible in the data flow.
Covers `.loc` assignment, chained indexing, and `inplace`.

**Rules:**
- `mut-loc-not-chained` - Assign with .loc, never with chained indexing
- `mut-no-inplace` - Avoid inplace=True

## Data Types (dtype)
**Impact:** HIGH

Make dtypes an explicit decision rather than an inference result.
Covers dtype declaration at load time, categorical columns, and NumPy dtype selection.

**Rules:**
- `dtype-explicit-io` - Specify dtypes when loading data
- `dtype-categorical` - Use category dtype for low-cardinality strings
- `dtype-numpy-explicit` - Choose NumPy dtypes explicitly

## Schema Validation (schema)
**Impact:** HIGH

Validate tabular data where it enters the system.
Covers pandera schemas as the DataFrame counterpart of Pydantic models.

**Rules:**
- `schema-pandera` - Validate DataFrames at boundaries with pandera

## Reproducibility (repro)
**Impact:** HIGH

Make experiments repeatable and free of hidden global state.
Covers explicit random number generators and seeding.

**Rules:**
- `repro-default-rng` - Use an explicit Generator for randomness

## Transformation Style (style)
**Impact:** MEDIUM

Write DataFrame transformations that read top-to-bottom as a single pipeline.
Covers method chaining, `assign`/`pipe`, and named aggregation.

**Rules:**
- `style-method-chaining` - Express transformations as method chains
- `style-named-aggregation` - Use named aggregation in groupby

## Typing (type)
**Impact:** MEDIUM

Type hints that carry dtype and column information.
Covers `numpy.typing`, `pandas-stubs`, and pandera-typed frames.

**Rules:**
- `type-array-annotations` - Annotate arrays and frames with precise types

## Testing (test)
**Impact:** MEDIUM

Assertions that fit floating-point arrays and DataFrames.
Covers `numpy.testing`, `pandas.testing`, and `pytest.approx`.

**Rules:**
- `test-numeric-assertions` - Compare numeric results with tolerance-aware assertions
