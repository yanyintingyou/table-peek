# table-peek

[简体中文](README.zh-CN.md) | English

table-peek lets an agent look at a huge table before it tries to analyze one.

A Python script reads the file and writes a few small artifacts. They tell the model what the table is, what one row means, where the data sits, and which parts stay uncertain. When the agent needs actual values, it pulls a bounded slice instead of loading the file.

## Why this exists

Hand a multi-GB CSV to a model and one of three things happens. The file blows the context window. It gets truncated. Or the model reads a few hundred raw rows and misses the structure.

The hard part is the first step: how is this file organized?

Three cases that break naive readers:

- An IMF BOP/PIP export puts a handful of descriptor columns next to hundreds of period columns, `1948` through `2025-Q4`, and stores observations inside them. `PUBLICATION_DATE` looks like a time column and is not.
- A World Bank workbook opens with a title, a note, and a merged cell block. The real header sits on row 5.
- An EPU country workbook uses `Year` and `Month` as keys and spreads 30+ countries across value columns.

Read those directly and `SCALE.ID` becomes a measure while the publication date becomes the observation period. table-peek makes those calls first and lists what needs your confirmation.

## What you get

Six files per run:

| File | Contents |
|---|---|
| `table_manifest.json` | File inventory, candidate tables or sheets, capability level, warnings |
| `table_profile.json` | Per column: type, examples, missing rate, statistics |
| `table_digest.md` | Structural summary for the model, answers "what does one row mean" |
| `ambiguities.md` | Open questions that need a human decision |
| `data_locator_spec.json` | Machine readable locator: entity, indicator, time coverage, observation key, value column |
| `data_locator_guide.md` | The same locator written out, with instructions for pulling values by country or indicator code |

Add `--extract-limit` and you also get `sample_extract.csv` with up to N real rows.

Most agents read `table_digest.md` and `ambiguities.md` and stop there. The locator matters when it is time to fetch values.

## Quick start

```bash
git clone https://github.com/yanyintingyou/table-peek.git
cd table-peek

python skills/data-science/table-peek/scripts/profile_tabular_file.py \
  -i your_file.csv \
  -o ./peek
```

Read these three first:

```text
./peek/table_digest.md
./peek/ambiguities.md
./peek/data_locator_guide.md
```

Sampling is the default. The script reads 1 MB to guess encoding and delimiter, scans at most 100k rows, and builds column profiles from the first 1,000 rows. A 20 GB file takes about as long as a 20 MB one: seconds.

### Exact numbers: `--full-scan`

```bash
python skills/data-science/table-peek/scripts/profile_tabular_file.py \
  -i your_file.csv -o ./peek --full-scan
```

This streams the whole file once and attaches `exact_stats` to every column: exact missing counts and rates, numeric min/max/mean, bounded distinct values, and a type inferred from every row. Memory stays flat. You pay one full read.

On a 31 MB CSV with 1,000,000 rows and 7 columns, a full scan plus an extract took about 34 seconds at 5.2 MB peak Python memory. Scale that to a few GB and you are budgeting hours, which is why sampling stays the default.

### Real rows: `--extract-limit`

```bash
python skills/data-science/table-peek/scripts/profile_tabular_file.py \
  -i your_file.csv -o ./peek \
  --extract-limit 20 \
  --extract-where COUNTRY.ID=US
```

Repeat `--extract-where` to AND conditions. Matching is exact after trimming. The script stops once it has enough rows and writes them to `sample_extract.csv`. This path supports CSV and TSV only.

### Sensitive files

```bash
# Mask emails, phone numbers, long digit strings
... --redact-samples

# Write no row or column values at all; extraction is disabled too
... --no-samples
```

`--no-samples` clears `samples`, `examples`, `top_values_sample`, and the sample-derived parts of the locator. Field names, types, and structural warnings stay, because you need those to understand the schema.

## What it recognizes

| Table shape | What the profiler reports |
|---|---|
| Simple time series | Time column, value column, row count, date range (FRED single-series CSV) |
| Long or panel table | Entity, time, categorical and numeric columns, likely row grain |
| SDMX / macro long table | Nine roles: entity, indicator, counterpart, frequency, time, value, unit/scale/status |
| IMF BOP/PIP wide time | Descriptor columns plus period columns, `value_layout: wide_time_columns`, reshape advice |
| Wide measure matrix | `Year` / `Month` / `Date` as keys, remaining columns as parallel measures |
| Report style Excel | Searches for the real header and data start, marks merged cells and note blocks |
| Lookup or classification | Code to label mapping, not forced into a time series |
| Matrix or cross tab | Flags both axes as carrying dimensions and asks for confirmation |
| PDF | File metadata only, never claims extracted tables |

When a table fits no known shape, the output says so and lists the specific questions in `ambiguities.md`. Treat the profiler labels as hypotheses, not verdicts.

## Install for your agent

The repo keeps two copies of the skill. Root `SKILL.md` and `skills/data-science/table-peek/SKILL.md` are identical, so edit both.

- **Claude Code**: copy `skills/data-science/table-peek/` to `~/.claude/skills/data-science/table-peek/`
- **OpenAI Codex**: read `AGENTS.md` inside the repo, or copy the skill to `~/.agents/skills/data-science/table-peek/`
- **OpenClaw**: copy to `<workspace>/skills/data-science/table-peek/` or the same path under `~/.openclaw/skills/`
- **Hermes Agent**: copy to `~/.hermes/skills/data-science/table-peek/`, then start a new session
- **Grok Build**: copy to `~/.grok/skills/table-peek/`
- **Generic AgentSkills loaders**: point at the directory holding `SKILL.md`; the name is `table-peek`

## Locator for macro data

For country by indicator by time panels from the IMF or SDMX, add `--domain-preset`:

```bash
... --domain-preset imf-bop
```

Values: `auto` (default), `generic`, `macro-timeseries`, `sdmx`, `imf-bop`.

Pass local codelists and the locator fills in labels. Without them it reports bare codes and says an external codelist is required:

```bash
... --domain-preset sdmx --indicator-mapping indicators.csv --entity-mapping areas.csv
```

Mapping files use columns like `code,label,description`.

To find out which indicators a multi-GB file actually contains, add `--domain-full-scan`. It writes `domain_indicator_catalog.json`, `.csv`, and `.md` with indicator coverage, time ranges, units, and a prefix-based classification of cross-border capital flow series.

## Keeping output small

`--output-policy auto` picks a level from file size and column count. It only shrinks the generated JSON. Your input file is never touched.

| Input | Policy | Effect |
|---|---|---|
| Under 10 MB, normal width | `full` | Every profiling detail kept |
| Medium file or 200+ columns | `balanced` | Fewer examples and top values |
| Over 100 MB or 1,000 columns | `compact` | Row samples dropped, role columns kept |
| Over 1 GB or 5,000 columns | `very-compact` | Detailed profiles capped near 300 columns |

Every threshold takes a flag, and `--output-policy full|balanced|compact|very-compact` overrides the automatic choice.

For a RAG or SQL pipeline downstream, add `--schema-preset generic|rag|sql`. That writes `downstream_schema_hint.json` as a starting point for field mapping.

## File types

| Format | Support | Notes |
|---|---|---|
| CSV / TSV / delimited text | Strong | Streaming, standard library; supports `--full-scan` and extraction |
| `.xlsx` / `.xlsm` | Practical | Unzips and reads the XML, detects sheets, used ranges, merged cells, candidate headers; no formulas, macros, or legacy `.xls` |
| PDF | Metadata | Conservative file level checks; no table extraction, no OCR |

The core script imports only `csv`, `zipfile`, `xml.etree`, `json`, and `re`, so Python 3.9+ runs it as is. The five packages in `requirements.txt` are optional extras for deeper analysis. The script never imports them; `--check-deps` only reports whether they exist.

## Known limits

- Default statistics come from a sample. Exact numbers need `--full-scan`, which takes hours on multi-GB files.
- `sample_extract.csv` holds the first N matches in file order. It is not a random sample and not a query engine. For filtered analysis across many keys, reshape the data and hand it to DuckDB or pandas.
- Excel extraction is not supported. `--extract-limit` applies to CSV and TSV.
- Dates in `.xlsx` can show up as Excel serial numbers because the script does not implement the style engine.
- Multi row headers, stacked sub-tables in one sheet, scanned PDFs, and formula driven workbooks still need a custom parser.
- The skill produces structural understanding. It does not analyze, clean, or modify your data.

## Development

```bash
python tests/test_profile_examples.py
```

Thirteen regression tests, standard library only. They cover FRED time series, IMF wide time columns, EPU wide measures, World Bank report headers, exact statistics, bounded extraction, privacy flags, and wide table truncation. GitHub Actions runs the same suite on Python 3.9 through 3.12.

When you change CLI behavior, privacy handling, or the output schema, update both `SKILL.md` copies, the matching file under `references/`, and `CHANGELOG.md`.

## License

MIT. Built by [yanyintingyou](https://github.com/yanyintingyou).
