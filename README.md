# DfPr

`DfPr` is a Rust library for loading CSV and Excel files into a Polars-backed dataframe, working with values through a small typed API, preparing columns, and converting numeric data to tensors from the sibling `Tensors` crate.

## Add the dependency

This repository is a local crate and currently depends on `../TZoR`:

```toml
[dependencies]
DfPr = { path = "../DfPr" }
```

Change the path to the location where you keep this checkout. The crate targets Rust 2024 edition.

## Quick start

```rust
use DfPr::dataframe::DataFrame;
use DfPr::types::{DataColumn, DataValue};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let frame = DataFrame::from_columns(vec![
        DataColumn::new("name", vec![
            DataValue::String("Ada".into()),
            DataValue::String("Linus".into()),
        ]),
        DataColumn::new("score", vec![
            DataValue::Float64(98.0),
            DataValue::Float64(91.0),
        ]),
    ])?;

    println!("shape: {:?}", frame.shape());
    println!("columns: {:?}", frame.cols());
    Ok(())
}
```

Most fallible operations return `DfPr::error::Result<T>`. `DataFrame` wraps Polars, and you can access the underlying Polars dataframe when you need an API not wrapped here.

## Guides

- [DataFrame API](docs/dataframe.md): constructing, selecting, transforming, joining, splitting, grouping, and tensor export.
- [File I/O](docs/io.md): supported formats, loading/saving, `LoadOptions`, and current option behavior.
- [Types](docs/types.md): `DataValue`, `DataColumn`, and `LoadOptions` syntax and semantics.
- [Preprocessing](docs/preprocessing.md): label encoding, category counts, one-hot encoding, and scaling.
- [Conversions and tensors](docs/conversion.md): Polars conversions, tensor shapes, supported types, and null handling.
- [Errors](docs/errors.md): result alias and error variants.

## Project layout

| Path | Purpose |
| --- | --- |
| `src/dataframe.rs` | Polars-backed `DataFrame` API and `GroupBy` wrapper |
| `src/io.rs` | CSV/Excel loading, CSV saving, and extension detection |
| `src/types.rs` | In-memory value, column, and load-option types |
| `src/conversion.rs` | Polars, `DataColumn`, and tensor conversions |
| `src/preprocessing.rs` | Label encoding and category frequency counting |
| `src/error.rs` | Library error type and result alias |

The implementation uses Polars 0.42, Calamine for XLSX reading, Chrono for date/time values, and the local `Tensors` crate. See `Cargo.toml` for the full dependency and feature list.

## Current scope

File detection supports `.csv`, `.xlsx`, and `.xls` extensions, but the Excel reader is implemented with Calamine's `Xlsx` reader. CSV loading currently delegates to Polars defaults; several `LoadOptions` fields are therefore not applied by CSV loading. The I/O guide calls out which options are active for each operation.

The API is an early-stage library (`0.1.0`). Consult the module guides for exact supported input types and limitations before relying on a conversion or loader option.
