# Python Data Science

NumPy and pandas best practices for data science and ML code, designed for AI agents and LLMs to write fast, correct, and reproducible numeric code.

## Overview

This skill provides 13 rules across 8 categories:

| Category | Prefix | Impact | Rules |
|----------|--------|--------|-------|
| Vectorization | `vec-` | CRITICAL | 2 |
| Indexing & Mutation | `mut-` | CRITICAL | 2 |
| Data Types | `dtype-` | HIGH | 3 |
| Schema Validation | `schema-` | HIGH | 1 |
| Reproducibility | `repro-` | HIGH | 1 |
| Transformation Style | `style-` | MEDIUM | 2 |
| Typing | `type-` | MEDIUM | 1 |
| Testing | `test-` | MEDIUM | 1 |

## Structure

```
skills/data-science/
├── SKILL.md                        # Skill overview with quick reference
├── metadata.json                   # Metadata (version, description)
├── README.md                       # This file
└── rules/
    ├── _sections.md                # Section definitions
    ├── _template.md                # Rule template
    ├── vec-no-python-loops.md      # Vectorize instead of loops / iterrows / apply
    ├── vec-no-incremental-growth.md# Build arrays and frames once
    ├── mut-loc-not-chained.md      # .loc assignment, no chained indexing
    ├── mut-no-inplace.md           # Avoid inplace=True
    ├── dtype-explicit-io.md        # Declare dtypes at load time
    ├── dtype-categorical.md        # category dtype for repeated labels
    ├── dtype-numpy-explicit.md     # Explicit NumPy dtypes
    ├── schema-pandera.md           # pandera schemas at boundaries
    ├── repro-default-rng.md        # Explicit np.random.Generator
    ├── style-method-chaining.md    # assign / pipe chains
    ├── style-named-aggregation.md  # Named aggregation in groupby
    ├── type-array-annotations.md   # numpy.typing and pandas-stubs
    └── test-numeric-assertions.md  # assert_allclose / assert_frame_equal
```

## Rules

### Vectorization (CRITICAL)
- `vec-no-python-loops` - Vectorize instead of Python loops, iterrows, and row-wise apply
- `vec-no-incremental-growth` - Build arrays and DataFrames once, not incrementally

### Indexing & Mutation (CRITICAL)
- `mut-loc-not-chained` - Assign with .loc, never with chained indexing
- `mut-no-inplace` - Avoid inplace=True

### Data Types (HIGH)
- `dtype-explicit-io` - Specify dtypes when loading data
- `dtype-categorical` - Use category dtype for low-cardinality strings
- `dtype-numpy-explicit` - Choose NumPy dtypes explicitly

### Schema Validation (HIGH)
- `schema-pandera` - Validate DataFrames at boundaries with pandera

### Reproducibility (HIGH)
- `repro-default-rng` - Use an explicit Generator for randomness

### Transformation Style (MEDIUM)
- `style-method-chaining` - Express transformations as method chains
- `style-named-aggregation` - Use named aggregation in groupby

### Typing (MEDIUM)
- `type-array-annotations` - Annotate arrays and frames with precise types

### Testing (MEDIUM)
- `test-numeric-assertions` - Compare numeric results with tolerance-aware assertions

## Related

- [coding-standards / validation-pydantic](../coding-standards/rules/validation-pydantic.md) - Scalar-record counterpart of `schema-pandera`
- [coding-standards / design-no-global-singleton](../coding-standards/rules/design-no-global-singleton.md) - Same principle behind `repro-default-rng`
- [testing / fixture-factory](../testing/rules/fixture-factory.md) - Building expected frames for numeric tests

## Usage

This skill is automatically applied when working with notebooks (`*.ipynb`, `notebooks/**`) and whenever the task involves numpy, pandas, DataFrames, or data pipelines.
