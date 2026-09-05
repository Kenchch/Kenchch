## Feng Jiang

**Data engineer in Christchurch, New Zealand. I build pipelines whose numbers can be audited.**

Open to data and analytics engineering roles in New Zealand.

My portfolio focuses on Python and SQL pipelines, data quality, and analytics delivery, with additional projects in distributed processing and applied machine learning.

## Start here

- **Data engineering:** [retail pipeline](https://github.com/Kenchch/retail-ai-pipeline) for orchestration, dbt and publication guarantees; [NZ pageviews](https://github.com/Kenchch/nz-attraction-pageviews#run-it) for a small offline demo of incremental ingestion and recovery.
- **Analytics:** [retail analysis](https://github.com/Kenchch/online-retail-analysis-r/blob/main/analysis.md) for SQL, credit-note reconciliation and business findings; [NZ sheep analysis](https://kenchch.github.io/nz-sheep-decline-by-region/) for a published report with source-data checks and uncertainty.
- **Applied ML:** [drone detection](https://github.com/Kenchch/aerial-small-object-detection) for measured accuracy and inference latency.

These are portfolio and study projects. Dataset totals and offline benchmarks
describe the analysed inputs and experiments, rather than revenue generated or
measured customer impact.

## Selected work

| Project | What stands out |
|---|---|
| [**retail-ai-pipeline**](https://github.com/Kenchch/retail-ai-pipeline) | End-to-end Airflow + dbt retail pipeline with atomic versioned publishing; **£10.25M gross valid sales** across 19,773 orders, with 522,566 loaded rows + 19,343 quarantined rows accounted for exactly. |
| [**aerial-small-object-detection**](https://github.com/Kenchch/aerial-small-object-detection) | YOLO11 small-object detection and tracking on VisDrone2019; Dockerized GPU workflow, ONNX export and measured accuracy/latency benchmarks over **38,759 validation boxes**. |
| [**nz-attraction-pageviews**](https://github.com/Kenchch/nz-attraction-pageviews) | Incremental Wikimedia API ingestion into DuckDB with idempotent recovery, offline validation and data-quality checks across Python 3.10–3.13. |
| [**online-retail-analysis-r**](https://github.com/Kenchch/online-retail-analysis-r) | Reproducible R + SQL analysis of **541,909 invoice lines**; cancellation-aware netting produces **£9.88M** in clean sales, backed by a SHA-256-pinned input and reconciliation audit. |

The two retail projects use the same UCI input but answer different accounting
questions. The Python pipeline reports valid positive sales before matching
credit notes; the R analysis nets matched sale/cancellation pairs and retains
exact duplicate rows. Their approximately £364k difference is reconciled in both project
READMEs.

## How I work

- **Reproducibility:** pinned or fingerprinted inputs, one-command rebuilds and committed evidence.
- **Auditability:** explicit quality gates, row-level reconciliation and immutable/atomic publication.
- **Engineering discipline:** CI, automated tests, documented limitations and measured—not assumed—performance.

## How this portfolio was built

I used AI pair-programming tools, including Claude Code and OpenAI Codex, for
drafting, refactoring and test scaffolding. I defined the problems, designed
the pipelines and data contracts, selected the quality rules, ran the
benchmarks on my own hardware, and reviewed and edited the resulting code.
Commits with assistant-contributed code retain their `Co-Authored-By` trailers.

## Stack

`Python` · `SQL` · `R` · `Airflow` · `dbt` · `DuckDB` · `pandas` · `PySpark` · `Docker` · `Power BI` · `PyTorch` · `ONNX`

[![retail-ai-pipeline CI](https://github.com/Kenchch/retail-ai-pipeline/actions/workflows/ci.yml/badge.svg)](https://github.com/Kenchch/retail-ai-pipeline/actions/workflows/ci.yml)
[![aerial detection CI](https://github.com/Kenchch/aerial-small-object-detection/actions/workflows/ci.yml/badge.svg)](https://github.com/Kenchch/aerial-small-object-detection/actions/workflows/ci.yml)
[![NZ pageviews CI](https://github.com/Kenchch/nz-attraction-pageviews/actions/workflows/ci.yml/badge.svg)](https://github.com/Kenchch/nz-attraction-pageviews/actions/workflows/ci.yml)
