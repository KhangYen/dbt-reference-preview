# dbt patterns I use on every project

A short, opinionated reference of the dbt patterns that come up **weekly** in production — not the tutorial material. Compiled from ~3 years of shipping dbt on BigQuery, Snowflake, and Postgres.

> **Why this repo exists:** I kept writing the same patterns from scratch on every project and looking up the same syntax. I organized them into a [printable 25-page reference pack](https://khangster587.gumroad.com/l/kowxqb) ($7.99) covering commands, SCD-2, incremental models, the 8 DQ tests every project needs, Jinja macros, materializations, and partition/cluster patterns.
>
> The patterns below are **free** — copy them into your project. If you want the full printable PDF with all 25 pages, that's the paid version linked above. No pressure.
>
> **Companion repo:** [sql-for-data-engineers-preview](https://github.com/KhangYen/sql-for-data-engineers-preview) — the same treatment for pipeline SQL (window functions, MERGE idioms, query plans, NULL handling) across Postgres/BigQuery/Snowflake/Databricks.

---

## 1. The idempotent incremental pattern (prevents duplicate rows)

The bug: your incremental model duplicates rows on backfills. The fix is `unique_key` + a lookback buffer:

```sql
{{
  config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge'
  )
}}

select *
from {{ source('stripe', 'charges') }}
{% if is_incremental() %}
  where created_at >= date_sub(_dbt_max_partition, interval 3 day)
{% endif %}
```

- `unique_key` makes the merge idempotent — re-running the same day doesn't double-write.
- The 3-day buffer catches late-arriving events. Tune to your latency profile: too small misses rows, too large reprocesses too much.

## 2. SCD Type 2 validation test (catches corrupt dimensions)

If you run SCD-2 snapshots, this singular test catches the corruption that breaks every downstream dimension — multiple `is_current = true` rows for the same entity:

```sql
-- tests/singular/assert_one_current_per_customer.sql
select customer_id, count(*) as n_current
from {{ ref('dim_customers') }}
where is_current = true
group by customer_id
having count(*) > 1
```

If this returns any rows, your close-out logic dropped a row somewhere. Run it on every SCD-2 dimension.

## 3. The Jinja macro I paste into every project

Safely convert amounts to USD with a fallback to the original if no FX rate is found:

```sql
{% macro convert_to_usd(amount_col, currency_col, rate_table='dim_fx_rates') %}
  coalesce(
    {{ amount_col }} * (
      select rate
      from {{ ref(rate_table) }}
      where currency = {{ currency_col }}
        and valid_from <= current_date
      order by valid_from desc
      limit 1
    ),
    {{ amount_col }}
  )
{% endmacro %}
```

Used in every revenue model I've ever shipped. Falls back to the original amount rather than nulling out revenue when FX data is missing.

## 4. The minimum viable `schema.yml` for any fact table

```yaml
models:
  - name: fct_charges
    description: "One row per charge event."
    columns:
      - name: charge_id
        tests:
          - unique
          - not_null
      - name: customer_id
        tests:
          - relationships:
              to: ref('dim_customers')
              field: customer_id
      - name: amount_usd
        tests:
          - dbt_utils.accepted_range:
              min_value: 0
```

Plus two singular tests that aren't in this YAML:
- **Row count > 0** — catches silent model failure (empty table after a bad filter).
- **No gaps in the date series** — for time-series facts, catches a day that didn't load.

## 5. The pre-merge checklist

Before merging any new analytics model:

- [ ] Filter on the partition column directly (no functions wrapping it — disables pruning)
- [ ] Every `<>` checked for the NULL-drop bug (`where x <> 'foo'` silently drops NULL rows)
- [ ] JOIN columns are non-null (or use `is distinct from`)
- [ ] Window functions have explicit frames (`rows between ...`)
- [ ] Aggregates that should ignore NULLs do; aggregates that shouldn't use `coalesce`
- [ ] Materialization choice matches access pattern (view vs table vs incremental)
- [ ] `EXPLAIN ANALYZE` doesn't show a surprise full scan
- [ ] Runs in < 30s on a representative partition

---

## What's in the full pack

The free patterns above are ~5 pages of a 25-page printable reference. The full pack also covers:

- **dbt project structure** that scales (the folder layout that survives a year)
- **Every CLI command** + the selector syntax that makes them powerful
- **Staging → marts pattern** with copy-paste SQL
- **SCD Type 2** done right — snapshots AND hand-rolled, with the trade-offs
- **Surrogate keys** with dbt-utils
- **All 8 DQ tests** every project needs (with YAML)
- **Jinja macro patterns** (3 macros, fully annotated)
- **Materializations decision table** — when to use view / table / incremental / ephemeral
- **Partition / cluster / bucket patterns** for BigQuery + Snowflake
- **Glossary** of dbt vocabulary

**Get the printable PDF (25 pages, $7.99):**

→ **[Gumroad](https://khangster587.gumroad.com/l/kowxqb)** — Pay-what-you-want above $7.99, includes 2-page content preview

→ **[Etsy](https://www.etsy.com/listing/4544349105/dbt-cheat-sheet-pdf-data-engineering)** — Instant download, printable on US Letter or A4

---

## Who this is for

- Data engineers and analytics engineers using dbt daily
- Analysts moving into analytics engineering
- Teams onboarding new hires (skip the 12-blog-post reading list)
- Interview prep — the patterns section covers what comes up

## License

The patterns in this repo are free to use (MIT-ish — copy them, no attribution required, no warranty). The printable PDF pack linked above is personal-use, one purchase = one user.

## Contributing

Found a pattern that should be here? Open an issue or PR. The goal is the shortlist of what *actually* comes up weekly, not an exhaustive reference.
