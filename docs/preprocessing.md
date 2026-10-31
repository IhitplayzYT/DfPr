# Preprocessing

Preprocessing APIs are split between the `preprocessing` module and convenience methods on `dataframe::DataFrame`.

## Label encoding

`LabelEncoder::fit(&DataColumn)` records distinct non-null values in first-seen order. Nulls are excluded from the classes. This ordering defines the code assignment:

```rust
use DfPr::preprocessing::LabelEncoder;
use DfPr::types::{DataColumn, DataValue};

let labels = DataColumn::new("animal", vec![
    DataValue::String("cat".into()),
    DataValue::String("dog".into()),
    DataValue::String("cat".into()),
    DataValue::Null,
]);
let encoder = LabelEncoder::fit(&labels);
assert_eq!(encoder.classes().len(), 2);
let codes = encoder.transform(&labels, "animal_id")?;
let restored = encoder.inverse_transform(&codes, "animal")?;
```

`transform(column, output_name)` emits zero-based `DataValue::Int32` codes. Null inputs remain null. Any non-null value absent from the fitted classes returns a conversion error. Fit the encoder on training data and reuse it for validation/test data to preserve a consistent mapping.

`inverse_transform(column, output_name)` accepts non-negative integer-valued numeric codes and restores their classes. Nulls remain null. Negative, fractional, non-numeric, or out-of-range codes return a conversion error. `classes()` provides a borrowed slice in encoding order.

For a dataframe, `frame.label_encode("animal", None)` returns a new frame and encoder, writing to `animal_encoded` by default. Pass `Some("animal_code")` to choose another output column. The original label column is retained.

## One-hot encoding

`DataFrame::one_hot_encode(column)` creates one boolean column for each distinct non-null category and removes the original column. Each generated name is `<source_column>_<category Display>`. A row has `true` in its matching category column; nulls are false in every generated column. If a generated name collides with an existing column, the method errors. Distinct categories that stringify to the same name can also cause a collision.

```rust
let expanded = frame.one_hot_encode("animal")?;
```

The lower-level `conversion::Onehot` helper is a legacy experimental function with implementation issues; prefer `DataFrame::one_hot_encode` for dataframe work. See [Conversions](conversion.md) before using it.

## Value counts

`preprocessing::value_counts(&DataColumn)` returns each distinct `DataValue` with its frequency in first-seen order. Null is counted as a category. `DataFrame::value_counts(column)` is the convenient fallible wrapper because it first validates and retrieves the named column.

## Numeric scaling

`DataFrame::Standardize(column)` computes population mean and standard deviation from the non-null values, then replaces values with `(x - mean) / std_dev`. Nulls remain null; if the standard deviation is zero, present values become `0.0`.

`DataFrame::MinMaxScale(column)` computes `(x - min) / (max - min)`. Nulls remain null; if min equals max, present values become `0.0`. Both methods error on all-null/empty data or non-numeric columns. Each returns a new frame and leaves the source frame alone.

These scaling methods calculate statistics from the supplied frame. For model evaluation, calculate transforms from training data only to avoid leaking test-set statistics.
