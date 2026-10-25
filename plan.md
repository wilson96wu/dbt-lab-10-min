# Plan: Re-create dbt-lab-10-min from scratch

Goal: rebuild a small dbt + DuckDB project that models a toy e-commerce dataset (customers, products, orders, payments): CSV seeds → staging views → mart tables, with tests.

Final result: 5 seeds, 4 staging views, 3 marts, 1 macro, 9 generic-test columns + 1 singular test. Everything runs with `uv run dbt build`.

---

## 0. Prerequisites

- [uv](https://docs.astral.sh/uv/) installed (`brew install uv`)
- Git
- Optional: DuckDB CLI (`brew install duckdb`) for inspecting the data

No database server is needed. DuckDB is embedded and writes to a local `dev.duckdb` file.

## 1. Create the folder and git repo

```bash
mkdir dbt-lab-10-min && cd dbt-lab-10-min
git init
```

## 2. Python environment with uv

```bash
uv init --bare            # creates pyproject.toml only
uv add dbt-duckdb         # installs dbt-core + duckdb adapter, writes uv.lock
```

Then edit `pyproject.toml` so it matches:

```toml
[project]
name = "dbt-lab-10-min"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "dbt-duckdb>=1.11.0",
]
```

## 3. `.gitignore`

```gitignore
target/
dbt_packages/
logs/
dbt_internal_packages/
*.duckdb
```

## 4. Folder skeleton

```bash
mkdir -p analyses macros models/staging models/marts seeds snapshots tests
touch analyses/.gitkeep macros/.gitkeep seeds/.gitkeep snapshots/.gitkeep tests/.gitkeep
```

## 5. `dbt_project.yml`

The project name **must match** the keys under `seeds:` and `models:`; otherwise the schema and materialization configs are silently ignored (see "Pitfalls" at the bottom).

```yaml
name: 'dbt_ecommerce'
version: '1.0.0'
config-version: 2

profile: 'default'

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

seeds:
  dbt_ecommerce:
    +schema: raw

models:
  dbt_ecommerce:
    staging:
      +materialized: view
      +schema: staging
    marts:
      +materialized: table
      +schema: marts
```

## 6. `profiles.yml` (project root)

dbt looks in the project root before `~/.dbt/`, so this keeps the lab self-contained.

```yaml
default:
  target: dev
  outputs:
    dev:
      type: duckdb
      path: dev.duckdb
      threads: 4
```

Verify:

```bash
uv run dbt debug
```

## 7. Seeds (`seeds/`)

Create these five CSV files exactly as shown.

**`seeds/raw_customers.csv`**

```csv
customer_id,first_name,last_name,email,city,country,created_at
1,Rahul,Sharma,rahul.sharma@email.com,Mumbai,India,2023-01-15
2,Priya,Patel,priya.patel@email.com,Delhi,India,2023-01-22
3,James,Wilson,james.wilson@email.com,New York,USA,2023-02-03
4,Ananya,Gupta,ananya.gupta@email.com,Bangalore,India,2023-02-14
5,Michael,Brown,michael.brown@email.com,London,UK,2023-03-01
6,Sneha,Reddy,sneha.reddy@email.com,Hyderabad,India,2023-03-10
7,Sarah,Johnson,sarah.johnson@email.com,San Francisco,USA,2023-03-18
8,Amit,Kumar,amit.kumar@email.com,Pune,India,2023-04-05
9,Emily,Davis,emily.davis@email.com,Chicago,USA,2023-04-12
10,Vikram,Singh,vikram.singh@email.com,Jaipur,India,2023-04-20
11,Olivia,Martinez,olivia.martinez@email.com,Toronto,Canada,2023-05-01
12,Deepak,Joshi,deepak.joshi@email.com,Chennai,India,2023-05-15
13,Sophia,Lee,sophia.lee@email.com,Sydney,Australia,2023-06-02
14,Arjun,Nair,arjun.nair@email.com,Kochi,India,2023-06-18
15,Daniel,Taylor,daniel.taylor@email.com,Berlin,Germany,2023-07-01
16,Kavya,Menon,kavya.menon@email.com,Kolkata,India,2023-07-14
17,Chris,Anderson,chris.anderson@email.com,Seattle,USA,2023-08-01
18,Neha,Verma,neha.verma@email.com,Lucknow,India,2023-08-20
19,Lucas,Garcia,lucas.garcia@email.com,Madrid,Spain,2023-09-05
20,Pooja,Iyer,pooja.iyer@email.com,Coimbatore,India,2023-09-22
```

**`seeds/raw_products.csv`**

```csv
product_id,product_name,category,price_in_cents,created_at
101,Wireless Mouse,Electronics,2499,2023-01-01
102,USB-C Hub,Electronics,3999,2023-01-01
103,Mechanical Keyboard,Electronics,7999,2023-01-01
104,Laptop Stand,Accessories,2999,2023-01-01
105,Monitor Light Bar,Accessories,4599,2023-01-01
106,Noise Cancelling Headphones,Electronics,12999,2023-01-01
107,Webcam HD 1080p,Electronics,5499,2023-01-01
108,Desk Mat XL,Accessories,1999,2023-01-01
109,Phone Stand,Accessories,999,2023-01-01
110,Portable Charger,Electronics,3499,2023-01-01
```

**`seeds/raw_orders.csv`**

```csv
order_id,customer_id,product_id,quantity,order_date,status
1001,1,101,2,2023-02-10,completed
1002,1,106,1,2023-03-15,completed
1003,2,103,1,2023-02-20,completed
1004,3,102,1,2023-03-05,completed
1005,3,108,2,2023-03-05,completed
1006,4,107,1,2023-03-18,completed
1007,5,106,1,2023-04-01,returned
1008,6,101,1,2023-04-10,completed
1009,6,104,1,2023-04-10,completed
1010,7,103,1,2023-04-22,completed
1011,8,110,2,2023-05-05,completed
1012,9,105,1,2023-05-12,completed
1013,10,109,3,2023-05-25,completed
1014,2,108,1,2023-06-01,completed
1015,1,104,1,2023-06-15,returned
1016,11,106,1,2023-06-20,completed
1017,12,102,2,2023-07-01,completed
1018,13,107,1,2023-07-10,completed
1019,14,101,1,2023-07-20,completed
1020,4,103,1,2023-08-01,completed
1021,15,110,1,2023-08-10,completed
1022,16,105,1,2023-08-25,completed
1023,17,106,1,2023-09-05,completed
1024,18,101,2,2023-09-15,completed
1025,19,108,1,2023-09-28,completed
```

**`seeds/raw_payments.csv`**

```csv
payment_id,order_id,payment_method,amount_in_cents,payment_date,status
2001,1001,credit_card,4998,2023-02-10,success
2002,1002,credit_card,12999,2023-03-15,success
2003,1003,upi,7999,2023-02-20,success
2004,1004,debit_card,3999,2023-03-05,success
2005,1005,credit_card,3998,2023-03-05,success
2006,1006,upi,5499,2023-03-18,success
2007,1007,credit_card,12999,2023-04-01,refunded
2008,1008,upi,2499,2023-04-10,success
2009,1009,upi,2999,2023-04-10,success
2010,1010,credit_card,7999,2023-04-22,success
2011,1011,debit_card,6998,2023-05-05,success
2012,1012,credit_card,4599,2023-05-12,success
2013,1013,upi,2997,2023-05-25,success
2014,1014,debit_card,1999,2023-06-01,success
2015,1015,credit_card,2999,2023-06-15,refunded
2016,1016,credit_card,12999,2023-06-20,success
2017,1017,upi,7998,2023-07-01,success
2018,1018,debit_card,5499,2023-07-10,success
2019,1019,upi,2499,2023-07-20,success
2020,1020,credit_card,7999,2023-08-01,success
2021,1021,debit_card,3499,2023-08-10,success
2022,1022,upi,4599,2023-08-25,success
2023,1023,credit_card,12999,2023-09-05,success
2024,1024,upi,4998,2023-09-15,success
2025,1025,debit_card,1999,2023-09-28,success
```

**`seeds/order_status_mapping.csv`** (loaded but not yet used by any model)

```csv
status_code,status_label,is_active_order
completed,Completed,false
returned,Returned,false
shipped,Shipped,true
processing,Processing,true
cancelled,Cancelled,false
```

Load and check:

```bash
uv run dbt seed
```

Expected: 5 seeds loaded. With the config in step 5, they land in the `main_raw` schema.

## 8. Macro

**`macros/cents_to_dollars.sql`**

```sql
{% macro cents_to_dollars(column_name) %}
    round({{ column_name }} / 100, 2)
{% endmacro %}
```

## 9. Sources

The source schema must be where the seeds were loaded. With `+schema: raw` on a DuckDB `main` default schema, that is `main_raw`.

**`models/staging/_sources.yml`**

```yaml
version: 2

sources:
  - name: raw
    schema: main_raw
    tables:
      - name: raw_customers
      - name: raw_orders
      - name: raw_products
      - name: raw_payments
```

## 10. Staging models (`models/staging/`)

Each one is a 1:1 cleaned view over a raw table: rename, cast, lowercase, convert cents to dollars.

**`stg_customers.sql`**

```sql
with source as (
    select * from {{ source('raw', 'raw_customers') }}
),

renamed as (
    select
        customer_id,
        first_name,
        last_name,
        first_name || ' ' || last_name as full_name,
        lower(email) as email,
        city,
        country,
        cast(created_at as date) as customer_created_at
    from source
)

select * from renamed
```

**`stg_orders.sql`**

```sql
with source as (
    select * from {{ source('raw', 'raw_orders') }}
),

renamed as (
    select
        order_id,
        customer_id,
        product_id,
        quantity,
        cast(order_date as date) as order_date,
        lower(status) as order_status
    from source
)

select * from renamed
```

**`stg_payments.sql`**

```sql
with source as (
    select * from {{ source('raw', 'raw_payments') }}
),

renamed as (
    select
        payment_id,
        order_id,
        payment_method,
        amount_in_cents,
        {{ cents_to_dollars('amount_in_cents') }} as amount,
        cast(payment_date as date) as payment_date,
        lower(status) as payment_status
    from source
)

select * from renamed
```

**`stg_products.sql`**

```sql
with source as (
    select * from {{ source('raw', 'raw_products') }}
),

renamed as (
    select
        product_id,
        product_name,
        category,
        price_in_cents,
        {{ cents_to_dollars('price_in_cents') }} as price,
        cast(created_at as date) as product_created_at
    from source
)

select * from renamed
```

## 11. Staging tests (`models/staging/_staging_models.yml`)

```yaml
version: 2

models:
  - name: stg_customers
    columns:
      - name: customer_id
        tests:
          - unique
          - not_null
      - name: email
        tests:
          - unique

  - name: stg_orders
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: order_status
        tests:
          - accepted_values:
              arguments:
                values: ['completed', 'returned', 'shipped', 'processing', 'cancelled']

  - name: stg_products
    columns:
      - name: product_id
        tests:
          - unique
          - not_null

  - name: stg_payments
    columns:
      - name: payment_id
        tests:
          - unique
          - not_null
      - name: payment_method
        tests:
          - accepted_values:
              arguments:
                values: ['credit_card', 'debit_card', 'upi']
```

Build the staging layer:

```bash
uv run dbt run --select staging
uv run dbt test --select staging
```

## 12. Mart models (`models/marts/`)

**`dim_customers.sql`**: one row per customer with `total_orders` and `lifetime_value` (sum of successful payments).

```sql
with customers as (
    select * from {{ ref('stg_customers') }}
),

orders as (
    select * from {{ ref('stg_orders') }}
),

payments as (
    select * from {{ ref('stg_payments') }}
),

customer_orders as (
    select
        customer_id,
        count(order_id) as total_orders
    from orders
    group by customer_id
),

customer_payments as (
    select
        orders.customer_id,
        sum(case when payments.payment_status = 'success' then payments.amount else 0 end) as lifetime_value
    from payments
    left join orders on payments.order_id = orders.order_id
    group by orders.customer_id
),

final as (
    select
        customers.customer_id,
        customers.full_name,
        customers.email,
        customers.city,
        customers.country,
        coalesce(customer_orders.total_orders, 0) as total_orders,
        coalesce(customer_payments.lifetime_value, 0) as lifetime_value
    from customers
    left join customer_orders on customers.customer_id = customer_orders.customer_id
    left join customer_payments on customers.customer_id = customer_payments.customer_id
)

select * from final
```

**`dim_products.sql`**: one row per product with `total_units_sold` and `total_revenue`.

```sql
with products as (
    select * from {{ ref('stg_products') }}
),

orders as (
    select * from {{ ref('stg_orders') }}
),

product_stats as (
    select
        product_id,
        sum(quantity) as total_units_sold
    from orders
    group by product_id
),

final as (
    select
        products.product_id,
        products.product_name,
        products.category,
        products.price,
        coalesce(product_stats.total_units_sold, 0) as total_units_sold,
        coalesce(product_stats.total_units_sold * products.price, 0) as total_revenue
    from products
    left join product_stats on products.product_id = product_stats.product_id
)

select * from final
```

**`fct_orders.sql`**: one row per order joined to customer, product and payment, with an `is_refunded` flag.

```sql
with orders as (
    select * from {{ ref('stg_orders') }}
),

payments as (
    select * from {{ ref('stg_payments') }}
),

products as (
    select * from {{ ref('stg_products') }}
),

customers as (
    select * from {{ ref('stg_customers') }}
),

final as (
    select
        orders.order_id,
        orders.order_date,
        orders.order_status,
        orders.quantity,
        customers.full_name as customer_name,
        products.product_name,
        products.price as unit_price,
        payments.payment_method,
        payments.amount as payment_amount,
        case when payments.payment_status = 'refunded' then true else false end as is_refunded
    from orders
    left join payments on orders.order_id = payments.order_id
    left join products on orders.product_id = products.product_id
    left join customers on orders.customer_id = customers.customer_id
)

select * from final
```

## 13. Singular test

**`tests/assert_payment_is_positive.sql`**: fails if any successful payment has an amount `<= 0`.

```sql
select
    payment_id,
    amount
from {{ ref('stg_payments') }}
where payment_status = 'success'
  and amount <= 0
```

## 14. Full build and verification

```bash
uv run dbt build          # seed + run + test in dependency order
```

Expected: 5 seeds, 4 views, 3 tables, and all tests pass.

Inspect the results:

```bash
uv run duckdb dev.duckdb "select * from main_marts.dim_customers limit 5"
# or without the CLI:
uv run python -c "import duckdb; print(duckdb.connect('dev.duckdb').sql('select * from main_marts.fct_orders'))"
```

Expected schemas in `dev.duckdb`: `main_raw` (seeds), `main_staging` (views), `main_marts` (tables).

Sanity checks:

- `dim_customers` has 20 rows; `dim_products` has 10; `fct_orders` has 25.
- Orders 1007 and 1015 have `is_refunded = true`.

Optional docs and lineage graph:

```bash
uv run dbt docs generate
uv run dbt docs serve
```

## 15. Commit

```bash
git add .
git commit -m "Initial dbt-duckdb e-commerce lab"
```

`.user.yml` is auto-generated by dbt (it holds an anonymous user id). It is safe to commit or ignore.

---

## Pitfalls (learned from the original repo)

1. **Project name must match the config keys.** The original repo has `name: 'my_new_project'` but keys the `seeds:`/`models:` blocks under `dbt_ecommerce`. dbt then ignores them, so no custom schemas or materializations are applied and everything lands in `main` as views. Keep them identical (this plan uses `dbt_ecommerce`).
2. **Source schema must match where seeds load.** DuckDB appends `+schema: raw` to the default `main`, giving `main_raw`. The original repo's latest commit sets the source schema to `main`, which works only because pitfall 1 prevents the `raw` schema from being applied. If you fix pitfall 1, change the source schema to `main_raw`. If you instead drop the `seeds:` `+schema`, use `main`.
3. **`order_status_mapping` is an unused seed.** It is a natural next step: join it into `stg_orders` or `fct_orders` to expose `status_label` and `is_active_order`.
4. **`accepted_values` syntax.** The `arguments:` wrapper is the newer dbt (1.10+) form. Older dbt versions expect `values:` directly under `accepted_values`.
