# MySQL vs DuckDB: OLTP-to-OLAP Benchmark

> Team project for COMP 8157 (Advanced Database Topics), University of Windsor, Summer 2026, supervised by Dr. Andreas S. Maniatis.
> Team 8: Rabiya Javed Farooq, Purab Singh Mohan, Nancy Bogati Collum, Sneha Gunturu.
> Original team repo: https://github.com/cyberwar18/COMP.-8157---Team-8
> Final report (IEEE format): [docs/Team8_Final_Report.pdf](docs/Team8_Final_Report.pdf)

**The question:** at what data size and query shape does a row-store database (MySQL) stop being good enough for analytics, compared to a column-store (DuckDB)?

We loaded the TPC-H benchmark dataset at three sizes (SF-1, SF-5, SF-10, up to about 60 million line items), transformed it into a star schema in both engines, then ran the same graduated query matrix against both and measured latency, loading time, storage, memory and CPU.

![MySQL vs DuckDB, multi group-by, full scan](results/chart_multi_group_by_full_scan.png)

## Key results

Multi-column GROUP BY with a full table scan (the heaviest query shape), on the transactional (OLTP) schema:

| | MySQL | DuckDB |
|---|---|---|
| SF-1 (about 6M line items) | 59.9 s | 0.32 s |
| SF-5 (about 30M line items) | 5,935 s (about 99 min) | 2.29 s |
| Slowdown from SF-1 to SF-5 (5x more data) | about 99x | about 7x |

**What this means in plain words:**
- At SF-5, DuckDB was **about 2,600x faster** than MySQL on heavy aggregation. MySQL's slowdown grew far faster than the data did, while DuckDB scaled close to linearly.
- **How the data is accessed mattered even more than the engine.** In MySQL, the same query went from about 99 minutes on a full scan to about 0.02 seconds as an indexed point lookup, roughly a 300,000x difference.
- **The star schema didn't always help.** It made DuckDB about 2.2 to 2.4x faster on single group-by and nested subqueries, but slightly slower on multi group-by. MySQL gained about 1.1x. The right design depends on the engine and the query.
- Building the star schema at SF-5 took 7.2 s in DuckDB and about 2.4 hours in MySQL. In MySQL it also shrank the on-disk footprint from about 8.18 GB to 6.66 GB.
- MySQL at SF-10 was not completed. The roughly 22 GB of raw data plus the engine's on-disk footprint went over our 108 GB machine during loading. The report documents this instead of estimating the missing results.

Numbers are medians across repeated runs, from `results/median_latency_summary.csv`, `results/transformation_metrics.csv` and the final report.

## My contributions (Nancy Bogati Collum)

**My role: lead contributor.** I coordinated the team (planning, task split, deadlines and merging), led the **query execution and metrics collection layer**, ran every benchmark, wrote the **final analysis and charts**, and authored the project documents. That's about 20 of the project's 32 commits.

- **Team coordination:** planned the phases, split tasks across the team, tracked deadlines and handled merging work into `main`.
- **Documentation:** wrote the pre-proposal, the full proposal (IEEE LaTeX + presentation deck) and the midterm status report, and co-authored the final report.

- **Query executor (`src/query_executor.py`):** runs the full query matrix across both engines, both schema layers, three scale factors and three index-access modes (full scan, indexed point lookup, range scan), and records latency, peak memory and CPU for every trial. I also owned the run configuration (`config.yaml`, `query_matrix.yaml`).
  - Fixed how index-mode filters get added to queries. Two earlier approaches (matching any WHERE, and matching the outermost WHERE) caused unresolvable and ambiguous column errors on nested subqueries. The final version targets the query block that actually scans the base fact table.
- **Smoke tests (`tests/test_smoke.py`):** quick checks that the config, schemas and query matrix are valid before starting runs that can take hours.
- **Ran the benchmarks:** all DuckDB runs (SF-1, SF-5, SF-10) and all MySQL runs (SF-1, SF-5). Single MySQL queries at SF-5 took over 90 minutes, so I backed up the results between runs.
- **Analysis and charts (`analysis.py`):** median latency summary plus charts for engine comparison, OLTP vs OLAP speedup, index-mode impact, latency vs scale (small multiples), DuckDB peak memory, and degradation rate normalized to SF-1.

**Tech:** Python, MySQL 8.0 (InnoDB), DuckDB, SQL (star schema, TPC-H), pandas, matplotlib, psutil, pytest, YAML config, Bash.

## More charts

| | |
|---|---|
| ![Degradation rate](results/chart_degradation_multi_group_by_full_scan.png) | ![Latency vs scale](results/chart_smallmultiples_oltp_full_scan.png) |
| ![Index mode impact](results/chart_indexmode_duckdb_multi_group_by_sf5.png) | ![DuckDB memory](results/chart_memory_duckdb_oltp_full_scan.png) |

---

*Full setup and run instructions from the original team README follow below.*



# OLTP-to-OLAP Transformation and Multi-Metric Performance Benchmarking: MySQL vs DuckDB


Rabiya Javed Farooq · Purab Singh Mohan · Nancy Bogati Collum · Sneha Gunturu

## What this project does

This project compares MySQL (row-oriented, OLTP-optimized) against DuckDB
(columnar, OLAP-optimized) across two dimensions most existing benchmarks skip:

1. **The transformation step itself.** We load raw TPC-H data in its native
   transactional form into both engines, then explicitly transform it into
   OLAP-ready structures (a star schema fact/dimension model plus a
   denormalized analytical view) identically in both engines and measure
   both layers separately.
2. **More than query latency.** Data loading time, on-disk storage footprint,
   memory consumption, CPU utilization, and query performance across
   aggregation types (single group-by, multi-column group-by, nested
   subqueries) at multiple scale factors (SF-1, SF-5, SF-10).

The goal is a practitioner-usable answer to: at what dataset scale and query
shape does each engine's architectural advantage break down?

## Repo structure

```
config/
  config.example.yaml     # copy to config.yaml and fill in your MySQL password
sql/
  mysql/, duckdb/          # oltp_schema.sql (Phase 1-2), olap_schema.sql (Phase 3)
queries/
  query_matrix.yaml        # graduated query matrix (Phase 5)
src/
  schema_loader.py          # Phase 1-2 — applies OLTP/OLAP DDL
  data_populator.py         # Phase 2 — bulk-loads TPC-H .tbl files
  transformation_executor.py # Phase 3 — builds star schema + wide view
  query_executor.py         # Phase 4-5 — runs the query matrix, captures metrics
  utils/
    db_connectors.py        # shared MySQL/DuckDB connection wrappers
    metrics.py               # CPU/memory sampling during query trials
analysis.py                 # Phase 6 — crossover thresholds + charts (run from repo root)
scripts/
  generate_tpch_data.sh     # builds/generates TPC-H .tbl files via dbgen
  run_all.sh                 # full pipeline, one scale factor, end to end
tests/
  test_smoke.py              # smoke tests for query executor prerequisites
data/
  sample/                    # small (5,000 rows/table) TPC-H sample at SF-1, SF-5, SF-10 — committed for grader inspection
  raw/                        # full generated .tbl files — git-ignored, see Dataset section below
results/
  raw_metrics.csv                       # full per-trial benchmark results
  raw_metrics_backup_20260718.csv       # timestamped backup
  median_latency_summary.csv            # clean summary table, produced by analysis.py
  transformation_metrics.csv            # transformation time + storage footprint
  chart_*.png                            # summary charts, produced by analysis.py
external/
  tpch-dbgen/                 # NOT included in the repo — clone manually, see Setup step 4
```

## Setup

**1. Install dependencies:**
```
pip install -r requirements.txt
```

**2. Set up MySQL 8.0** locally, and create the database:
```sql
CREATE DATABASE tpch;
```

**3. Configure credentials:**
```
cp config/config.example.yaml config/config.yaml
```
Edit `config/config.yaml` with your real MySQL password. This file is
git-ignored — never commit it.

**4. Generate TPC-H data.** Requires a C compiler (gcc/make). On Windows,
use WSL, Git Bash, or a Linux server (e.g. your university's shared server)
rather than fighting native compilation:
```bash
git clone https://github.com/electrum/tpch-dbgen.git external/tpch-dbgen
cd external/tpch-dbgen
cp makefile.suite Makefile
# edit Makefile: CC = gcc, DATABASE = SQLSERVER, MACHINE = LINUX, WORKLOAD = TPCH
make
cd ../..
bash scripts/generate_tpch_data.sh 1
```
This drops `.tbl` files into `data/raw/sf1/`. Repeat with `5` and `10` as the
argument for the other scale factors. If you generated data on a remote
server, copy the `.tbl` files into that same local path (scp/sftp).

`external/tpch-dbgen` is intentionally **not** committed to this repository
(the generator itself, plus its compiled binaries, are excluded via
`.gitignore`) — always clone it fresh per the command above rather than
expecting it to appear after `git clone` of this repo.

## Dataset

This project uses the industry-standard TPC-H benchmark dataset at scale
factors SF-1, SF-5, and SF-10, generated via the official `tpch-dbgen` tool
(see Setup step 4 above).

- **`data/sample/`** (committed to this repo) — a 5,000-row-per-table sample
  of each table at each scale factor, so the data shape and structure can be
  inspected directly without downloading the full dataset.
- **Full dataset** (too large for GitHub — SF-10's `lineitem.tbl` alone is
  ~7.8GB, well over GitHub's 100MB per-file limit). Not hosted online.
- Alternatively, regenerate the full dataset locally at any scale factor
  using the commands in Setup step 4.

## Running the pipeline

Each phase can be run independently, or all at once via `run_all.sh`.

**Phase 1-2 — schema + data:**
```bash
python -m src.schema_loader --engine both --schema oltp
python -m src.data_populator --engine both --scale-factor 1
```

**Phase 3 — OLTP-to-OLAP transformation:**
```bash
python -m src.transformation_executor --engine both
```
Builds `fact_lineitem_orders`, `dim_customer`, `dim_supplier`, `dim_part`,
`dim_date`, and `analytical_wide_view` on both engines. Logs transform time
and storage footprint delta to `results/transformation_metrics.csv`.

**Phase 4-5 — query matrix + metrics:**
```bash
python -m src.query_executor --engine both --schema-layer both
```

**Phase 6 — analysis:**
```bash
python3 analysis.py
```
Reads `results/raw_metrics.csv` and produces `results/median_latency_summary.csv`
plus the `results/chart_*.png` summary charts.

**Or all phases at once, for a given scale factor:**
```bash
bash scripts/run_all.sh 1
```

## Current status

- [x] Phase 1-2: OLTP schema + TPC-H SF-1 data loaded (MySQL + DuckDB)
- [x] Phase 3: OLTP-to-OLAP transformation complete (MySQL + DuckDB)
- [x] Phase 4-5: query matrix execution
  - [x] DuckDB, SF-1, OLTP + OLAP: complete
  - [x] DuckDB, SF-5, OLTP + OLAP: complete
  - [x] DuckDB, SF-10, OLTP + OLAP: complete
  - [x] MySQL, SF-1, OLTP + OLAP: complete
  - [x] MySQL, SF-5, OLTP + OLAP: complete (reduced repetitions — see benchmark
        results documentation, §2)
  - [ ] MySQL, SF-10, OLTP + OLAP: **not completed** — disk-capacity and
        bulk-load-duration limitation, documented in the benchmark results
        documentation, §5. Not a harness failure.
- [x] Phase 6: crossover analysis + charts

## Notes for contributors

- Windows users: `run_all.sh` and `generate_tpch_data.sh` are bash scripts —
  run them via WSL or Git Bash, not plain PowerShell.
- MySQL's `LOAD DATA LOCAL INFILE` needs `allow_local_infile=True` on the
  client connection and `local_infile=1` server-side — already handled in
  `db_connectors.py`, but worth knowing if you're debugging connection issues.
- TPC-H's `.tbl` files end each line with a trailing delimiter, which some
  naive CSV readers misinterpret as an extra column — also already handled
  in `db_connectors.py`'s DuckDB loader.
- `data/raw/`, `*.duckdb`, and `config/config.yaml` are git-ignored on
  purpose — regenerate full-scale data locally rather than trying to commit
  it (`lineitem.tbl` alone is ~750MB at SF-1 and ~7.8GB at SF-10, well over
  GitHub's file size limit). A small sample is committed at `data/sample/`
  for inspection — see the Dataset section above.
- `results/` **is** tracked and committed (not git-ignored) — it contains
  the benchmark output (`raw_metrics.csv`, `median_latency_summary.csv`,
  charts) that graders need to see.
