# Value and column types

The `types` module defines the library's owned, row-oriented interface for individual values and columns, plus the configuration type used by file I/O.

## `DataValue`

```rust
pub enum DataValue {
    Null,
    Bool(bool),
    Int8(i8), Int16(i16), Int32(i32), Int64(i64),
    UInt8(u8), UInt16(u16), UInt32(u32), UInt64(u64),
    Float32(f32), Float64(f64),
    String(String),
    Date(chrono::NaiveDate),
    DateTime(chrono::NaiveDateTime),
    Time(chrono::NaiveTime),
    Vec(Vec<f64>),
}
```

Construct values explicitly, for example `DataValue::Int32(4)`, `DataValue::String("ok".into())`, or `DataValue::Null`. The date/time variants use Chrono's naive (timezone-free) types. `Vec` stores a vector of `f64` and is used by some conversion helpers; it is not a scalar numeric value.

Methods:

- `is_numeric()` is true for the integer and float variants only.
- `is_tensorizable()` is true for `Vec` only.
- `as_f64()` converts numeric variants to `f64`; other variants return `None`.
- `type_name()` returns a short string such as `"i32"`, `"string"`, `"datetime"`, or `"vec"`.
- `id(&mut DefaultHasher)` hashes an internal byte representation and is used by legacy encoding helpers. It is not a stable external identifier format; do not persist it as a category ID.
- `Display` renders the value for formatting. `Null` renders as `null`; strings render without quotes.

The enum derives `PartialEq`, not `Eq` or `Hash`. Floating-point comparisons therefore follow Rust `f32`/`f64` semantics (for example, NaN is not equal to itself).

## `DataColumn`

```rust
pub struct DataColumn {
    pub name: String,
    pub values: Vec<DataValue>,
}
```

Create columns with `DataColumn::new(name, values)`. Both fields are public, so callers can inspect and modify them directly. A dataframe's columns must have matching lengths.

```rust
let ages = DataColumn::new("age", vec![
    DataValue::Int32(29),
    DataValue::Null,
    DataValue::Int32(34),
]);
assert_eq!(ages.len(), 3);
assert!(ages.is_numeric());
let as_f64: Vec<Option<f64>> = ages.as_numeric_f64()?;
```

Methods:

- `len()` and `is_empty()` inspect the number of entries.
- `dtype()` returns the first entry's type name, or `"unknown"` for an empty column. It does not validate all entries.
- `is_numeric()` returns true when every value is a number or null. Empty columns pass this check vacuously.
- `is_tensorizable()` returns true when every value is a `Vec` or null. Empty columns also pass vacuously.
- `as_numeric_f64()` returns one `Option<f64>` per row, retaining null positions. It errors if any non-null value is non-numeric.

When converting a `DataColumn` to Polars, the first non-null value chooses the series type; values of another variant are converted to null by the current converter. Keep non-null values homogeneous. An all-null column becomes a nullable integer series, and an empty column is represented as a one-null integer series by the current implementation; avoid relying on empty-column round trips.

## `LoadOptions`

```rust
#[derive(Clone, Debug, Default)]
pub struct LoadOptions {
    pub has_header: bool,
    pub separator: Option<char>,
    pub skip_rows: usize,
    pub null_values: Option<Vec<String>>,
    pub dtype_overrides: Option<Vec<(String, String)>>,
}
```

Builder syntax is chainable:

```rust
let options = LoadOptions::new()
    .with_header(true)
    .with_separator(';')
    .with_skip_rows(1)
    .with_null_values(vec!["NA".into(), "-".into()])
    .with_dtype_overrides(vec![("age".into(), "i64".into())]);
```

Fields are public and builders consume and return `Self`. See [File I/O](io.md) for which operations currently honor these values. At present, CSV loading does not apply the options; Excel loading only uses `skip_rows` and `separator`; CSV saving uses `has_header` and `separator`. `null_values` and `dtype_overrides` are currently not applied by these implementations.
