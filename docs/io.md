# File I/O

The `io` module provides free functions for loading CSV/XLSX data, saving CSV, and detecting a file type from its extension. The dataframe wrapper exposes `from_csv`, `from_excel`, and `from_file` convenience constructors.

## Loading

```rust
use DfPr::dataframe::DataFrame;
use DfPr::types::LoadOptions;

let options = LoadOptions::new();
let csv = DataFrame::from_csv("data/measurements.csv", &options)?;
let excel = DataFrame::from_excel("data/measurements.xlsx", &options)?;
let detected = DataFrame::from_file("data/measurements.csv", &options)?;
```

Free function signatures:

```rust
pub fn load_csv<P: AsRef<Path>>(path: P, options: &LoadOptions) -> Result<polars::frame::DataFrame>
pub fn load_excel<P: AsRef<Path>>(path: P, options: &LoadOptions) -> Result<polars::frame::DataFrame>
pub fn load_file<P: AsRef<Path>>(path: P, options: &LoadOptions) -> Result<polars::frame::DataFrame>
```

CSV uses `File::open` and `polars::prelude::CsvReader::finish`. Excel loading reads the first worksheet with Calamine's `Xlsx` reader, converts cells to strings, joins them into CSV text, and parses that text using Polars. The Excel path uses `skip_rows` and `separator`; all cells are stringified before Polars infers the resulting CSV columns.

`detect_ftype(path)` is based on the lower-cased extension. It recognizes `csv`, `xlsx`, and `xls` and returns `FileType::{Csv, Excel}`. Although `xls` is recognized, the current reader type is `Xlsx`; legacy `.xls` support is not guaranteed and may fail during workbook opening. Files without extensions and other extensions return a conversion error.

## Loading options and current behavior

`LoadOptions` fields and builder methods are described in [Types](types.md). The current implementation applies options inconsistently:

| Option | `load_csv` / `load_csv_sync` | `load_excel` |
| --- | --- | --- |
| `has_header` | Ignored; Polars reader defaults are used | Ignored by the intermediary CSV reader |
| `separator` | Ignored; Polars reader defaults are used | Used to join cells into intermediary CSV text |
| `skip_rows` | Ignored | Skips rows in the first worksheet before parsing |
| `null_values` | Ignored | Ignored |
| `dtype_overrides` | Ignored | Ignored |

In particular, do not assume setting `with_header(false)` changes CSV loading; this option currently only affects CSV output. The Excel separator influences how the temporary CSV string is built and is not equivalent to a full Excel import setting.

## Synchronous/buffered aliases

`load_csv_sync` reads the entire file into a UTF-8 `String` before parsing. Despite its name, it is still a synchronous function. `load_excel_sync` delegates to `load_excel`, and `load_file_sync` dispatches to these functions. The CSV sync function currently ignores `LoadOptions` just like `load_csv`.

```rust
use DfPr::io::{load_file_sync, FileType, detect_ftype};
use DfPr::types::LoadOptions;

let kind = detect_ftype("data/records.csv")?;
assert_eq!(kind, FileType::Csv);
let raw_polars_df = load_file_sync("data/records.csv", &LoadOptions::new())?;
```

## Saving CSV

```rust
use DfPr::io::save_csv;
use DfPr::types::LoadOptions;

save_csv(frame.as_polars(), "out.csv", &LoadOptions::new().with_header(true))?;
```

`save_csv(&polars_df, path, &options)` creates or truncates the destination and writes with the configured separator (comma by default) and `include_header(options.has_header)`. It accepts a Polars dataframe, so use `frame.as_polars()` with the wrapper. File and Polars failures are returned through the library `Result`.

## Errors

I/O errors (including file open/create failures) become `DataloaderError::Io`; Polars parsing/writing failures become `Polars`; unsupported/missing extensions and missing worksheets use `Conversion`; an empty worksheet after skipping rows uses `EmptyData`. See [Errors](errors.md).
