# polars-lab

Working repo for practicing Polars dataframe operations. Sub-projects live in
`projects/`, one folder each; datasets go in `data/` (CSV/parquet/JSON files
are gitignored, so the repo tracks structure only).

`GUIDE.md` is the working reference: environment setup with uv, visidata for
terminal data exploration, and a Polars cheat sheet covering CSV/parquet IO,
filtering, derived columns, group-by aggregation, and lazy evaluation with
`pl.scan_csv` — written against the mortgage dataset described below.

## Setup

    uv sync    # creates .venv/ and installs polars (Python >= 3.14)
    uv run python projects/<project>/<script>.py

## Data

The examples use the RMBS mortgage panel dataset (622,489 rows, 50,000
borrowers over 60 periods) from creditriskanalytics.net (Baesens, Roesch,
Scheule, "Credit Risk Analytics", Wiley 2016). It is not committed; download
mortgage.csv from that site into `data/` before running the examples.

## Layout

    pyproject.toml   dependencies (managed with uv)
    GUIDE.md         setup notes and Polars reference
    data/            datasets (contents gitignored)
    projects/        one folder per sub-project
