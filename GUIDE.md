# polars-lab — Startup Guide

Quick reference for getting this project running on Arch Linux.

## First-time setup (fresh clone)

```bash
# Clone the repo
git clone git@github.com:ddayfinsci-cell/polars-lab.git
cd polars-lab

# Install uv if not present
sudo pacman -S uv

# Create venv and install dependencies (one command)
uv sync
```

That's it. `uv sync` reads `pyproject.toml`, creates `.venv/`, and installs polars.

## Day-to-day usage

```bash
cd ~/polars-lab

# Run any script through uv (auto-activates the venv)
uv run python projects/my_script.py

# Or activate the venv manually
source .venv/bin/activate
python projects/my_script.py

# Add a new dependency
uv add matplotlib    # plotting
uv add pyarrow       # parquet I/O, arrow interop
uv add hvplot        # interactive plots
```

## Project layout

```
polars-lab/
├── pyproject.toml        # Dependencies live here
├── .venv/                # Virtual env (gitignored)
├── data/                 # Datasets (CSVs/parquet gitignored)
│   └── mortgage.csv      # 622K rows — RMBS securitization portfolio
├── projects/             # Sub-projects, one folder each
│   └── my_analysis/
│       └── explore.py
└── GUIDE.md              # This file
```

## Getting data in

### Read CSV
```python
import polars as pl

df = pl.read_csv("data/mortgage.csv")
```

### Read from URL
```python
import polars as pl

df = pl.read_csv("https://example.com/data.csv")
```

### Read parquet (faster for large files)
```python
df = pl.read_parquet("data/loans.parquet")
```

### Convert CSV to parquet (do this once, then use parquet)
```python
pl.read_csv("data/mortgage.csv").write_parquet("data/mortgage.parquet")
# Parquet is ~5-10x smaller and much faster to load
```

## Viewing data

```python
import polars as pl
df = pl.read_csv("data/mortgage.csv")

df.shape              # (rows, cols)
df.schema             # column names and types
df.head(10)           # first 10 rows
df.describe()         # summary stats for all columns
df.glimpse()          # transposed view — one row per column, fits terminal
df.sample(5)          # random 5 rows

# Specific columns
df.select("balance_time", "FICO_orig_time", "default_time")
df.select(pl.col("^.*orig.*$"))   # regex column selection
```

## Core operations cheat sheet

### Filter
```python
# Loans that defaulted
defaults = df.filter(pl.col("default_time") == 1)

# High LTV loans with low FICO
risky = df.filter(
    (pl.col("LTV_orig_time") > 80) & (pl.col("FICO_orig_time") < 650)
)
```

### Sort
```python
df.sort("balance_time", descending=True)
```

### New columns
```python
df = df.with_columns(
    (pl.col("balance_time") / pl.col("balance_orig_time")).alias("balance_ratio"),
    (pl.col("interest_rate_time") - pl.col("Interest_Rate_orig_time")).alias("rate_change"),
)
```

### Group and aggregate
```python
# Default rate by FICO bucket
df.with_columns(
    ((pl.col("FICO_orig_time") / 50).floor() * 50).alias("fico_bucket")
).group_by("fico_bucket").agg(
    pl.col("default_time").mean().alias("default_rate"),
    pl.col("balance_orig_time").mean().alias("avg_balance"),
    pl.len().alias("count"),
).sort("fico_bucket")
```

### Lazy evaluation (use for large data)
```python
# Lazy = builds a query plan, executes once at .collect()
result = (
    pl.scan_csv("data/mortgage.csv")
    .filter(pl.col("LTV_orig_time") > 80)
    .group_by("investor_orig_time")
    .agg(pl.col("default_time").mean().alias("default_rate"))
    .sort("default_rate", descending=True)
    .collect()
)
```

## The mortgage dataset

**Source:** creditriskanalytics.net — RMBS securitization portfolio data
from International Financial Research.

- **622,489 rows** — monthly performance observations
- **50,000 unique borrowers** tracked over 60 periods
- Panel data: each row is one borrower at one time period

### Key columns

| Column | Meaning |
|---|---|
| `id` | Borrower ID |
| `time` | Observation period |
| `orig_time` | Origination period |
| `balance_time` | Current outstanding balance |
| `balance_orig_time` | Balance at origination |
| `LTV_time` | Current loan-to-value ratio |
| `LTV_orig_time` | LTV at origination |
| `interest_rate_time` | Current interest rate |
| `FICO_orig_time` | FICO score at origination |
| `hpi_time` | House price index (current) |
| `hpi_orig_time` | House price index (at origination) |
| `gdp_time` | GDP growth rate |
| `uer_time` | Unemployment rate |
| `REtype_SF_orig_time` | 1 = single-family property |
| `REtype_CO_orig_time` | 1 = condo |
| `REtype_PU_orig_time` | 1 = PUD (planned unit development) |
| `investor_orig_time` | 1 = investor property (not owner-occupied) |
| `default_time` | **1 = loan defaulted this period** |
| `payoff_time` | 1 = loan paid off this period |
| `status_time` | Terminal status flag |

### Quick analysis starters

```python
import polars as pl
df = pl.read_csv("data/mortgage.csv")

# How many unique borrowers?
df.select(pl.col("id").n_unique())

# Overall default rate
df.filter(pl.col("default_time") == 1).height / df.select("id").n_unique()

# Default rate by property type
df.filter(pl.col("time") == pl.col("time").max()).group_by(
    "REtype_SF_orig_time", "REtype_CO_orig_time", "REtype_PU_orig_time"
).agg(
    pl.col("default_time").mean().alias("default_rate"),
    pl.len().alias("n"),
)

# Balance amortization for a single borrower
df.filter(pl.col("id") == 1).select("time", "balance_time").sort("time")
```

## Polars vs Pandas — key differences

| Concept | Pandas | Polars |
|---|---|---|
| Select columns | `df[["a", "b"]]` | `df.select("a", "b")` |
| Filter rows | `df[df.x > 5]` | `df.filter(pl.col("x") > 5)` |
| New column | `df["new"] = ...` | `df.with_columns(...)` |
| Groupby | `df.groupby("a").mean()` | `df.group_by("a").agg(pl.col("b").mean())` |
| Chaining | Awkward | Native — chain everything |
| Index | Row index is central | No index — use columns |
| Mutability | Mutable by default | Immutable — returns new frames |
| Lazy eval | No | `pl.scan_csv()` + `.collect()` |

## Useful links

- [Polars User Guide](https://docs.pola.rs/)
- [Polars API Reference](https://docs.pola.rs/api/python/stable/reference/)
- [Polars Expressions](https://docs.pola.rs/user-guide/expressions/)
- [Dataset citation: Baesens, Roesch, Scheule — Credit Risk Analytics (Wiley, 2016)](http://www.creditriskanalytics.net/)
