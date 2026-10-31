# Changelog

All notable changes to this project are documented here.

## 1.3.0 — 2026-10-31

### Changed

- Renamed the skill from `tabular-file-understanding` to `table-peek`. The packaged copy now lives at `skills/data-science/table-peek/`, and the `name:` field, Cursor rule, CI paths, AGENTS/CLAUDE instructions, and OpenAI metadata all follow it. Installs created before 1.3.0 still work, but they sit under the old directory name; replace them when convenient.
- Rewrote `README.md` and `README.zh-CN.md`: what the tool does for an agent, how to run the three common modes, and what it will not do. Repeated full command paths collapsed into one clone-and-run block.

## 1.2.0 — 2026-10-31

### Added

- `--extract-limit N` plus repeatable `--extract-where COL=VALUE` write at most N matching rows to `sample_extract.csv`, so an agent can inspect real values from a huge delimited file without dumping it or writing ad-hoc parsing code.
- `columns[].exact_stats` for `--full-scan`: exact missing count/rate, numeric min/max/mean, bounded distinct values, and a type inferred from every row. Streamed, so memory stays flat.
- `table_digest.md` gained "Exact Statistics (all rows)" and "Bounded Extract" sections.
- `tables[].sample_extract` and `profiling_options.extract_limit` / `extract_where` in `table_manifest.json`.

### Fixed

- `wide_measure_columns` no longer requires more than 30 columns; it now triggers on measure ratio (≥6 columns, ≥60% measures, no period columns). The EPU fixture is classified correctly instead of falling back to `unknown`.
- `wide_time_series_table` no longer requires ≥12 columns, so fiscal-year headers such as `FY89…FY92` are recognized as period columns even in small sheets.
- `--full-scan` previously left column statistics sample-bounded while only counting rows; it now produces exact statistics.
- `--redact-samples` / `--no-samples` now also suppress `exact_stats.top_values`.
- `SKILL.md` version field (was `1.0.0`) now matches the profiler `VERSION`.

### Improved

- Hot paths tightened: numeric detection skips `float()` exceptions for non-numeric cells, and date detection runs a cheap digit gate before regexes. A 1,000,000-row × 7-column full scan dropped from ~82 s to ~34 s on the same machine, at 5.2 MB peak Python memory.
- Regression suite expanded from 9 to 13 tests (exact statistics, bounded extract, unknown filter column, wide-measure detection).

## 1.1.1 — 2026-05-24

### Fixed

- Made `--no-samples` remove both row-level samples and column-level value samples (`examples` and `top_values_sample`) from `table_profile.json`, and suppress sample-derived values in locator artifacts.
- Made duplicate or blank CSV headers unique for stable downstream JSON/profile handling.
- Marked short-code first rows such as `US,JP,CN` as medium-confidence/ambiguous headers instead of high-confidence headers.
- Added explicit `profiling_success` and `understanding_quality` fields so invalid/empty workbooks are not mistaken for successful profiling.
- Removed duplicate unreachable logic in `guess_shape_type()`.
- Passed the pre-policy manifest into `build_data_locator()` instead of an empty placeholder.

### Improved

- Compact/very-compact output policy now prioritizes inferred entity/indicator/time/value/unit/scale/status columns and wide period columns instead of blindly keeping only the first N columns.
- Output-policy redaction now propagates the `--redact-samples` setting during secondary truncation.
- Added regression coverage for privacy, duplicate headers, ambiguous header rows, invalid XLSX files, and wide files with important columns near the tail.
- Added GitHub Actions CI for Python 3.9–3.12.
- Expanded README documentation on installation, output examples, and Excel/PDF/privacy limits.

## 1.1.0 — Initial public release

- Python-standard-library profiler for CSV/TSV, `.xlsx`/`.xlsm`, and metadata-first PDF profiling.
- Agent Skills-compatible layout under `skills/data-science/table-peek/`.
- Domain-aware locator for macro/SDMX/IMF BOP-like files.
- Adaptive output policies for large and wide files.
- Synthetic examples and regression tests for FRED, IMF wide-time, EPU wide-measure, and World Bank report-style layouts.
