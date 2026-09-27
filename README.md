# Feng Jiang

Data and analytics engineering · Christchurch, New Zealand.
Open to data and analytics engineering roles.
[Email](mailto:janfinq@gmail.com) · [LinkedIn](https://www.linkedin.com/in/fengjch)

**Start here → [retail-ai-pipeline](https://github.com/Kenchch/retail-ai-pipeline)**

A year of a UK online retailer's invoices, turned into sales reporting whose
totals trace back to the source file, and "bought together" recommendations.
Of 541,909 invoice lines, 3.6% are quarantined with a stated reason rather than
silently dropped. The recommendations reach a 56.3% hit-rate against 52.8% for
a most-popular baseline. Airflow, dbt, DuckDB, SQLite and Power BI.
[Design notes](https://github.com/Kenchch/retail-ai-pipeline/blob/main/docs/DESIGN.md) ·
[Run it](https://github.com/Kenchch/retail-ai-pipeline#run-it) ·
[Power BI pages](https://github.com/Kenchch/retail-ai-pipeline/blob/main/bi/README.md)

<a href="https://github.com/Kenchch/retail-ai-pipeline/blob/main/bi/README.md"><img src="https://raw.githubusercontent.com/Kenchch/retail-ai-pipeline/main/bi/screenshots/page1-sales.png" width="800" alt="Power BI sales overview: revenue, orders, average order value and customers, with revenue by month against the previous year, by country and by product"></a>

## Other projects

- [nz-attraction-pageviews](https://github.com/Kenchch/nz-attraction-pageviews) — How much online attention eight New Zealand attractions get each day, collected nightly from Wikipedia. A day that fails or arrives incomplete is retried or held back, never silently lost: an offline replay of 720 rows and then 24 more ends with zero duplicate (venue, day) pairs. Python + DuckDB.
- [online-retail-analysis-r](https://github.com/Kenchch/online-retail-analysis-r) — Same retail dataset, analysed in R + SQL. Every dropped row is logged with a reason and totals are checked against a SHA-pinned copy of the source, so the numbers can be traced; its net revenue matches the pipeline's to the penny.
- [nz-sheep-decline-by-region](https://github.com/Kenchch/nz-sheep-decline-by-region) — Where New Zealand's sheep went, by region, 2002 to 2025: the flock fell 41%. Stats NZ suppresses small cells, so the analysis reconciles regional totals against national ones and accounts for the gaps. R + Quarto, [published report](https://kenchch.github.io/nz-sheep-decline-by-region/).

## Outside the data track

- [aerial-small-object-detection](https://github.com/Kenchch/aerial-small-object-detection) — A benchmarking exercise: the same reproducibility habits (a pinned checkpoint with its SHA-256, stubbed tests, ONNX parity checks) applied to a computer-vision model. Kept for the methodology, not the domain. mAP50 0.375 on validation and 0.318 on held-out test-dev; ONNX on CUDA 7–26% faster than eager PyTorch across four sessions.

Archived, kept because they still reproduce rather than because they are current: PySpark coursework implementations for [Million Song](https://github.com/Kenchch/Million-Song-Dataset-Analysis-with-Spark) and [GHCN-Daily](https://github.com/Kenchch/GHCN-Daily-Climate-Analysis-with-PySpark). Each says so at the top, with the reason its Spark pin will not move.


[![retail CI](https://github.com/Kenchch/retail-ai-pipeline/actions/workflows/ci.yml/badge.svg)](https://github.com/Kenchch/retail-ai-pipeline/actions/workflows/ci.yml)
[![pageviews CI](https://github.com/Kenchch/nz-attraction-pageviews/actions/workflows/ci.yml/badge.svg)](https://github.com/Kenchch/nz-attraction-pageviews/actions/workflows/ci.yml)
[![R CI](https://github.com/Kenchch/online-retail-analysis-r/actions/workflows/ci.yml/badge.svg)](https://github.com/Kenchch/online-retail-analysis-r/actions/workflows/ci.yml)
[![Reproduce analysis](https://github.com/Kenchch/nz-sheep-decline-by-region/actions/workflows/reproduce.yml/badge.svg)](https://github.com/Kenchch/nz-sheep-decline-by-region/actions/workflows/reproduce.yml)
