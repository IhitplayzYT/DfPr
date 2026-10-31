# Errors

Fallible APIs use the crate alias:

```rust
pub type Result<T> = std::result::Result<T, DataloaderError>;
```

The error enum is `DfPr::error::DataloaderError`:

| Variant | Meaning |
| --- | --- |
| `Io(std::io::Error)` | File open/read/write or I/O-backed parsing failure |
| `Polars(polars::error::PolarsError)` | Polars operation or parsing/writing failure |
| `Tensor(String)` | Tensor construction or tensor-library failure |
| `Conversion(String)` | Invalid input, unsupported conversion, or value validation failure |
| `ColumnNotFound(String)` | A named column was not present |
| `InvalidDataType(String, String, String)` | A value/column had a type other than the requested type |
| `UnsupportedOperation(String)` | Operation or source dtype is not implemented |
| `EmptyData` | An operation requires nonempty input |
| `ShapeMismatch(String)` | Inputs do not have compatible dimensions |

Errors implement `std::error::Error` and are formatted with contextual prefixes such as `IO error:`, `Polars error:`, and `Conversion error:`. Use `?` to propagate them into a compatible error type, or pattern-match when callers need to distinguish cases:

```rust
use DfPr::error::{DataloaderError, Result};

fn read_score(frame: &DfPr::dataframe::DataFrame) -> Result<()> {
    match frame.column("score") {
        Ok(scores) => println!("{} values", scores.len()),
        Err(DataloaderError::ColumnNotFound(name)) => eprintln!("missing column: {name}"),
        Err(error) => return Err(error),
    }
    Ok(())
}
```

The exact variant depends on the API path: some methods explicitly map missing names to `ColumnNotFound`, while methods delegated directly to Polars can return `Polars` instead.
