# dbt-lab-10-min

A small dbt project that models a toy e-commerce dataset (customers, products, orders, payments). It's a hands-on lab: load CSV seeds, clean them in a staging layer, and build analytics-ready dimension and fact tables on top, with tests along the way.

## Project structure

```
.
├── dbt_project.yml
├── macros/
│   └── cents_to_dollars.sql      # round(col / 100, 2)
├── models/
│   ├── staging/                  # 1:1 cleaned views over raw tables
│   │   ├── _sources.yml
│   │   ├── _staging_models.yml   # schema tests
│   │   ├── stg_customers.sql
│   │   ├── stg_orders.sql
│   │   ├── stg_payments.sql
│   │   └── stg_products.sql
│   └── marts/                    # business-facing tables
│       ├── dim_customers.sql
│       ├── dim_products.sql
│       └── fct_orders.sql
├── seeds/                        # sample CSV data
│   ├── raw_customers.csv
│   ├── raw_orders.csv
│   ├── raw_payments.csv
│   ├── raw_products.csv
│   └── order_status_mapping.csv
└── tests/
    └── assert_payment_is_positive.sql
```

## Data flow

```
seeds (raw_*.csv) ──► staging (stg_*, views) ──► marts (dim_*/fct_*, tables)
```

| Layer | Model | Description |
|-------|-------|-------------|
| Staging | `stg_customers`, `stg_orders`, `stg_payments`, `stg_products` | Rename, cast and normalise raw columns (e.g. lower-cased order status, cents converted to dollars via the `cents_to_dollars` macro). |
| Marts | `dim_customers` | One row per customer with `total_orders` and `lifetime_value` (sum of successful payments). |
| Marts | `dim_products` | One row per product with `total_units_sold` and `total_revenue`. |
| Marts | `fct_orders` | One row per order joined to customer, product and payment, including an `is_refunded` flag. |

## Tests

- **Generic tests** (`models/staging/_staging_models.yml`): `unique` / `not_null` on primary keys, `unique` on customer email, and `accepted_values` for order status (`completed`, `returned`, `shipped`, `processing`, `cancelled`) and payment method (`credit_card`, `debit_card`, `upi`).
- **Singular test** (`tests/assert_payment_is_positive.sql`): fails if any successful payment has an amount `<= 0`.

## Getting started

### Prerequisites

- [uv](https://docs.astral.sh/uv/) (manages Python and dependencies)
- DuckDB is used as the warehouse via `dbt-duckdb`; no separate database server is needed

### 1. Set up the environment with uv

On a fresh clone (once `pyproject.toml` and `uv.lock` are committed):

```bash
uv sync
```

To set the project up from scratch:

```bash
uv init --bare          # creates pyproject.toml without sample files
uv add dbt-duckdb       # installs dbt-core + duckdb, writes uv.lock
```

Commit `pyproject.toml` and `uv.lock`.

### 2. Configure the DuckDB profile

`dbt_project.yml` uses `profile: 'default'`. Put this in `~/.dbt/profiles.yml`, or in a `profiles.yml` at the project root (dbt checks the project root first):

```yaml
default:
  target: dev
  outputs:
    dev:
      type: duckdb
      path: dev.duckdb
      threads: 4
```

Add `*.duckdb` to `.gitignore` so the database file isn't committed.

### 3. Fix the project config

See [Known issues / setup notes](#known-issues--setup-notes): rename the project to `dbt_ecommerce` and set the source schema to `main_raw`.

### 4. Run it

Prefix commands with `uv run` so they use the project environment:

```bash
uv run dbt debug         # verify the profile and connection
uv run dbt deps          # only needed if packages are added
uv run dbt seed          # load the CSVs into DuckDB
uv run dbt run           # build staging views and mart tables
uv run dbt test          # run all tests
uv run dbt build         # seed + run + test in dependency order
```

Run a subset with selectors, e.g. `uv run dbt run --select staging` or `uv run dbt build --select +dim_customers`.

Browse the docs and lineage graph:

```bash
uv run dbt docs generate
uv run dbt docs serve
```

### Inspect the data

```bash
uv run duckdb dev.duckdb     # needs the DuckDB CLI, e.g. brew install duckdb
uv run python -c "import duckdb; print(duckdb.connect('dev.duckdb').sql('select * from main_marts.dim_customers'))"
```

## Known issues / setup notes

These are things to fix when adapting the lab to your environment:

1. **Project name mismatch.** `dbt_project.yml` is named `my_new_project`, but the `seeds:` and `models:` config blocks are keyed under `dbt_ecommerce`. As a result the configured schemas (`raw`, `staging`, `marts`) and materializations are **not applied**. Rename the project to `dbt_ecommerce`, or change those keys to `my_new_project`.
2. **Source schema.** `models/staging/_sources.yml` points at schema `learndbt` (marked `CHANGE THIS`). With DuckDB, `dbt seed` loads into `main_raw` (the `+schema: raw` config is appended to the default `main` schema), so set the source schema to `main_raw`, or drop `+schema: raw` and use `main`. Otherwise staging models won't find their sources.
3. **Unused seed.** `order_status_mapping.csv` is loaded but not yet referenced by any model.
