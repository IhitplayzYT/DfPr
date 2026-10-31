# Conversions and tensors

The `conversion` module translates between Polars `Series`, `DataColumn`, and the local `Tensors` crate. Most functions return `DfPr::error::Result`.

## Polars Series and `DataColumn`

```rust
use DfPr::conversion::{data_column_to_series, series_to_data_column};
use DfPr::types::{DataColumn, DataValue};
use polars::prelude::Series;

let input = Series::new("count", &[Some(2_i32), None, Some(5)]);
let column = series_to_data_column(&input)?;
let output = data_column_to_series(&DataColumn::new(
    "label",
    vec![DataValue::String("yes".into()), DataValue::Null],
))?;
```

`series_to_data_column(&Series)` supports Polars null, bool, signed/unsigned integer, float, string, date, datetime, and time dtypes. Unsupported Series dtypes return `UnsupportedOperation`. Null entries become `DataValue::Null`.

`data_column_to_series(&DataColumn)` selects the Polars type from the first non-null `DataValue` and supports all scalar variants above plus `Vec`. Values with mismatching variants become null. `Vec` is represented as a nested Series. Empty columns currently convert to a one-null `i32` Series; all-null columns become all-null `i32` Series.

## 1D numeric tensors

The tensor functions are generic over `Tensors::dtype::Element`; values are converted through `T::from_f64`.

- `data_column_to_tensor::<T>(&DataColumn)` accepts numeric DataValues only and errors on nulls.
- `series_to_tensor::<T>(&Series)` supports Polars integer and float types listed in the implementation and errors on null or non-numeric values.
- `DataFrame::column_as_tensor::<T>(name)` is the wrapper method.

```rust
use DfPr::conversion::data_column_to_tensor;
use DfPr::types::{DataColumn, DataValue};
use Tensors::dtype::F32;

let values = DataColumn::new("x", vec![DataValue::Float64(1.5), DataValue::Float64(2.0)]);
let tensor = data_column_to_tensor::<F32>(&values)?;
```

Use an `Element` type available from the sibling `Tensors` crate. The generic type describes output storage; conversion is via `f64`, so very large integers may lose precision.

## 2D tensors

- `data_columns_to_tensor_2d::<T>(&[DataColumn])` requires at least one column, equal column lengths, numeric data, and no nulls.
- `series_to_tensor_2d::<T>(&[Series])` also requires nonempty, equal-length columns and no nulls. Its current Polars support is specifically `Int64`, `Float64`, `Float32`, and `Int32`; other numeric dtypes return a conversion error.
- `DataFrame::columns_as_tensor_2d::<T>(&["x", "y"])` uses the Series converter.
- `DataFrame::numeric_as_tensor_2d::<T>()` finds numeric columns and then uses that same Series converter. It can therefore fail if selected columns have unsupported dtypes or null entries.

The 2D layout is row-major with shape `(row_count, selected_column_count)`. The code appends values row by row and builds a tensor from those rows. Empty input columns are rejected by the free conversion functions. With a dataframe containing zero rows but at least one selected column, tensor construction behavior depends on the `Tensors` crate and may fail.

```rust
let features = frame.columns_as_tensor_2d::<F32>(&["height", "weight"])?;
let all_numeric = frame.numeric_as_tensor_2d::<F32>()?;
```

## Legacy categorical helpers

`Onehot(&DataColumn)` and `Ordinal(&DataColumn)` are public but currently unsafe to call successfully: both allocate a vector with capacity and then index it without pushing/resizing, which panics for nonempty input. Their category ordering also comes from a `HashSet`, so it is not a stable mapping. They are documented here so they are discoverable, but should not be used; use `DataFrame::one_hot_encode` or `LabelEncoder` instead.

`col_to_tensor<T: Float>(&DataColumn)` returns `Tensor<f64>` despite its generic parameter. It supports numeric columns (dropping nulls) and columns of `Vec` values. Because numeric nulls are filtered out, output length can be shorter than input length; mixed or non-tensorizable values return an error. Prefer the explicit 1D/2D APIs when null preservation and shape matter.
