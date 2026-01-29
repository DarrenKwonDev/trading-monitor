# Trading Monitor - Notes (QuestDB + Grafana)

This file captures the current decisions and operating assumptions from the
initial planning conversation.

## Goal

- Run QuestDB + Grafana via Docker.
- Ingest near-real-time data into QuestDB.
- Visualize the data in Grafana (tables and time-series charts).
- Minimize solo-operator cognitive load and development time.

## Key Decisions

- Stack: QuestDB + Grafana.
- Ingestion: InfluxDB Line Protocol (ILP) into QuestDB.
- Visualization: Grafana queries QuestDB (typically via Postgres wire protocol).

## How Chart Type Is Chosen (Grafana)

- Chart type is primarily a Grafana panel choice plus the shape of the query
  result.
- If the query returns a `time` column + numeric fields, Grafana naturally
  treats it as time-series (line/area/bar over time).
- If the query returns categorical breakdowns (e.g., symbol/exchange), bar
  charts and transformations (group/pivot) are common.
- For raw event inspection, use table panels.

## "Schema-less" Expectations

- Grafana is not a datastore; you generally do not "push JSON into Grafana".
- QuestDB is columnar and not fully schema-less. Some schema exists.
- ILP can auto-create tables/columns, which is close to "minimal DDL".

### Recommended Minimal Modeling Pattern

To keep flexibility while staying Grafana-friendly:

- Option A (metrics / long format):
  - One table with minimal stable columns, e.g. `ts`, `metric`, `value`, plus
    a few tags like `symbol`, `exchange`, `strategy`.
  - Adding new metrics typically does not require schema changes (new `metric`
    values only).

- Optional raw retention:
  - A separate raw events table storing the original payload as text (JSON
    string) for debugging/reprocessing.

### ILP Practical Rules

- Keep tag cardinality low: use tags only for common filters (symbol/exchange,
  not unique IDs like orderId).
- Keep types stable: avoid sending a field as string sometimes and number
  other times.
- Avoid "column explosion": do not map highly variable payload keys into many
  ILP fields; keep such payloads in a raw text column/table.

## Product Risk: Vendor/Company Shutting Down

- Even if the company disappears, open-source software plus pinned Docker
  images means the system does not immediately stop working.
- The long-term risk is maintenance/updates.
- Mitigations:
  - Pin QuestDB and Grafana image versions (avoid `latest`).
  - Keep ingestion and queries on standard surfaces (ILP + SQL/Postgres).
  - Periodically export key tables to CSV/Parquet for migration optionality.

## Host Resource Check (This Machine)

- CPU: 16 vCPU (Ryzen 7 1700).
- RAM: ~15.6 GiB total.
- Disk:
  - SSD: `/dev/sda2` mounted at `/` (~58G free).
  - HDD: `/dev/sdb1` mounted at `/home/drkup/code` and `/mnt/hdd` (~429G free).

Conclusion: resources are sufficient for QuestDB + Grafana.

## Where Data Will Be Stored

- Anything stored under `/home/drkup/code/trading-monitor` lives on the HDD
  (`/dev/sdb1`), not the SSD.

## Preference on Storage

- Prefer storing QuestDB/Grafana Docker volumes on HDD.
- Expect potentially higher latency vs SSD; keep dashboards focused on recent
  windows and use aggregated views if needed.
