**Data engineering / data science — I build reproducible, auditable pipelines where every published number can be traced back to its source.**

Based in Christchurch, New Zealand. My projects sit at the intersection of data quality, analytics engineering, machine learning and practical delivery.

## Selected work

| Project | What stands out |
|---|---|
| [**retail-ai-pipeline**](https://github.com/Kenchch/retail-ai-pipeline) | End-to-end Airflow + dbt retail pipeline with atomic versioned publishing; **66 Python tests**, **£10.25M reconciled** across 19,773 orders, and 522,566 loaded rows + 19,343 quarantined rows accounted for exactly. |
| [**aerial-small-object-detection**](https://github.com/Kenchch/aerial-small-object-detection) | YOLO11 small-object detection and tracking on VisDrone2019; **130 tests**, Dockerized GPU workflow, ONNX export and measured accuracy/latency benchmarks over **38,759 validation boxes**. |
| [**nz-attraction-pageviews**](https://github.com/Kenchch/nz-attraction-pageviews) | Incremental Wikimedia API ingestion into DuckDB with idempotent recovery and data-quality checks; **187 offline tests** across Python 3.10–3.13. |
| [**online-retail-analysis-r**](https://github.com/Kenchch/online-retail-analysis-r) | Reproducible R + SQL analysis of **541,909 invoice lines**; cancellation-aware netting produces **£9.88M** in clean sales, backed by a SHA-256-pinned input and reconciliation audit. |
| [**Million-Song-Dataset-Analysis-with-Spark**](https://github.com/Kenchch/Million-Song-Dataset-Analysis-with-Spark) | PySpark ML over **48.4M listening events** and **12.2 GB of audio features**; genre classification plus implicit-feedback ALS recommendations, with sanitized source notebooks and CI. |
| [**GHCN-Daily-Climate-Analysis-with-PySpark**](https://github.com/Kenchch/GHCN-Daily-Climate-Analysis-with-PySpark) | Distributed climate-data engineering for the **13+ GB GHCN-Daily archive**; fixed-width station enrichment, New Zealand temperature trends and global precipitation outputs. |

## How I work

- **Reproducibility:** pinned or fingerprinted inputs, one-command rebuilds and committed evidence.
- **Auditability:** explicit quality gates, row-level reconciliation and immutable/atomic publication.
- **Engineering discipline:** CI, automated tests, documented limitations and measured—not assumed—performance.

## Stack

`Python` · `SQL` · `R` · `Airflow` · `dbt` · `DuckDB` · `pandas` · `PySpark` · `Docker` · `Power BI` · `PyTorch` · `ONNX`

[![retail-ai-pipeline CI](https://github.com/Kenchch/retail-ai-pipeline/actions/workflows/ci.yml/badge.svg)](https://github.com/Kenchch/retail-ai-pipeline/actions/workflows/ci.yml)
[![aerial detection CI](https://github.com/Kenchch/aerial-small-object-detection/actions/workflows/ci.yml/badge.svg)](https://github.com/Kenchch/aerial-small-object-detection/actions/workflows/ci.yml)
[![NZ pageviews CI](https://github.com/Kenchch/nz-attraction-pageviews/actions/workflows/ci.yml/badge.svg)](https://github.com/Kenchch/nz-attraction-pageviews/actions/workflows/ci.yml)
[![Million Song CI](https://github.com/Kenchch/Million-Song-Dataset-Analysis-with-Spark/actions/workflows/ci.yml/badge.svg)](https://github.com/Kenchch/Million-Song-Dataset-Analysis-with-Spark/actions/workflows/ci.yml)
[![GHCN Daily CI](https://github.com/Kenchch/GHCN-Daily-Climate-Analysis-with-PySpark/actions/workflows/ci.yml/badge.svg)](https://github.com/Kenchch/GHCN-Daily-Climate-Analysis-with-PySpark/actions/workflows/ci.yml)
