# DataFrame API

`dataframe::DataFrame` is a cloneable wrapper around `polars::frame::DataFrame`. Operations generally return a new wrapper, leaving the original frame unchanged. Mutable access is available through `as_mut_polars`.

```rust
use DfPr::dataframe::DataFrame;
use DfPr::error::Result;
use DfPr::types::{DataColumn, DataValue};

fn make_frame() -> Result<DataFrame> {
    DataFrame::from_columns(vec![
        DataColumn::new("city", vec![
            DataValue::String("Delhi".into()),
            DataValue::String("Pune".into()),
        ]),
        DataColumn::new("temperature", vec![
            DataValue::Float64(32.0),
            DataValue::Float64(27.0),
        ]),
    ])
}
```

## Construction and Polars access

- `DataFrame::from_columns(Vec<DataColumn>) -> Result<DataFrame>` converts each column to a Polars Series. All columns must satisfy Polars' dataframe shape and naming rules.
- `DataFrame::from_polars(polars_df)` wraps an existing Polars dataframe.
- `as_polars(&self)` and `as_mut_polars(&mut self)` expose shared or mutable access to the inner frame.
- `from_csv(path, &LoadOptions)`, `from_excel(path, &LoadOptions)`, and `from_file(path, &LoadOptions)` are convenience constructors. See [File I/O](io.md) for the behavior of options.

## Shape and columns

- `height()`, `width()`, and `shape()` return row count, column count, and `(rows, columns)`.
- `cols()` returns column names in dataframe order.
- `column(name)` returns a converted `DataColumn`; `columns(names)` returns several in requested order; `as_data_columns()` converts every column.
- `select(&["a", "b"])` returns a frame with named columns.
- `with_column(name, column)` adds or replaces a column using the name carried by `DataColumn`. Its `_name` parameter is currently ignored. `add_column` and `add_col` are aliases with the same behavior.
- `drop(&["a", "b"])` removes columns; `rename(old, new)` renames one; `head(n)`, `tail(n)`, and `slice(offset, length)` select rows.

```rust
let first_two = frame.select(&["city", "temperature"])?;
let city = first_two.column("city")?;
let preview = first_two.head(5)?;
let without_city = first_two.drop(&["city"])?;
```

Unknown column names return `DataloaderError::ColumnNotFound` or a wrapped Polars error depending on the method.

## Filtering and mapping

`filter` evaluates a closure once per row. Its argument is a row slice containing `DataValue`s ordered by `cols()`:

```rust
let warm = frame.filter(|row| {
    matches!(row.get(1), Some(DataValue::Float64(t)) if *t >= 30.0)
})?;
```

This convenience path materializes every row and converts every column, so it is best for modest data. `filter_expr(expr)` delegates filtering to a Polars lazy expression:

```rust
use polars::prelude::{col, lit};

let warm = frame.filter_expr(col("temperature").gt(lit(30.0)))?;
```

`map_column(name, f)` maps `Fn(&DataValue) -> DataValue` over one existing column and replaces it. Use a consistent output `DataValue` variant (with `Null` as needed), because the first non-null value determines the Polars series dtype.

## Sorting, grouping, aggregation, and joins

- `sort(column, descending)` sorts ascending when `false` and descending when `true`.
- `group_by(&[column, ...])` returns `GroupBy`. Available aggregations are `first`, `last`, `sum`, `count`, `mean`, `min`, and `max`.
- `aggregate(&[Expr])` applies a Polars lazy aggregation expression.
- `inner_join`, `left_join`, `right_join`, and `full_join` join on one same-named key. `cross_join` produces a Cartesian product.
- `join(other, on, JoinType)` dispatches to those methods. `on` is required except for `JoinType::Cross`.
- `concat_vertical` appends rows and expects compatible schemas; `concat_horizontal` combines columns side by side.

```rust
let by_city = frame.group_by(&["city"])?.mean()?;
let ordered = frame.sort("temperature", true)?;
let deduplicated = frame.unique(Some(&["city"]))?;
```

## Nulls and category inspection

- `null_count()` returns a dataframe containing null counts by column.
- `drop_nulls()` removes rows containing nulls.
- `fill_null(FillNullStrategy)` delegates to Polars.
- `unique(subset)` keeps the first row for each unique key (or the full row if `None`).
- `unique_values(column)` returns distinct values in first-seen order, including `DataValue::Null` if present.
- `value_counts(column)` returns `(value, count)` pairs in first-seen order, including null.
- `categorical_columns()` returns columns containing at least one string, bool, date, datetime, or time. It does not classify every non-numeric Polars type as categorical.

## Splitting data

`train_test_split(test_size, shuffle, seed)` returns `(train, test)`. `test_size` must be finite and strictly between 0 and 1. Test rows are `ceil(row_count * test_size)`, clamped so both outputs have at least one row. Shuffling uses a deterministic internal generator; omitted seed means seed `0`. With `shuffle == false`, the first rows are assigned to the test partition and the remaining rows to train.

`stratified_train_test_split(test_size, stratify_by, seed)` shuffles row indexes within each label category and rounds each category's test count, clamped to keep at least one row per category in each partition. It errors if any category appears fewer than twice. `split_by_mask(test_mask)` returns `(train, test)` and expects exactly one bool per row; a wrong mask length propagates a Polars error.

## Preprocessing helpers

- `label_encode(column, output_name)` returns `(new_frame, encoder)`. Default output name is `<column>_encoded`.
- `one_hot_encode(column)` drops the source column and creates bool columns named `<column>_<display value>`. Nulls produce false in all dummy columns. Existing output-name collisions are errors.
- `Standardize(column)` replaces the column with population z-scores. Nulls remain null; a constant column becomes zero. The method uses a capitalized name in the current API.
- `MinMaxScale(column)` replaces numeric values with values in `[0, 1]`. Nulls remain null; a constant column becomes zero.

Scaling errors if a column has no non-null values or contains non-numeric values. These methods fit their statistics on the entire column passed to them; for machine-learning evaluation, split first and derive training statistics separately if data leakage matters.

## Tensor export

- `column_as_tensor::<T>(name)` converts a numeric column to a 1D `Tensors::tensor::Tensor<T>`.
- `columns_as_tensor_2d::<T>(&[...])` converts named numeric columns to a 2D tensor, rows by columns.
- `numeric_as_tensor_2d::<T>()` selects columns that `DataColumn::is_numeric` considers numeric and converts them in dataframe order.

Conversions require supported numeric Polars dtypes and do not accept null cells. See [Conversions and tensors](conversion.md).

## Return values and errors

Most transforms return `crate::error::Result<DataFrame>`. Polars failures are wrapped in `DataloaderError::Polars`; absent columns can produce `ColumnNotFound`; validation and unsupported cases commonly use `Conversion` or `UnsupportedOperation`. See [Errors](errors.md).
