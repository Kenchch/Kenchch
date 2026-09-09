# Feng Jiang

Data and analytics engineering · Christchurch, New Zealand.
Open to data and analytics engineering roles. [Email](mailto:janfinq@gmail.com).

**Start here → [retail-ai-pipeline](https://github.com/Kenchch/retail-ai-pipeline)**

Airflow + dbt pipeline over 541,909 retail rows: a quality gate and quarantine
table, atomic versioned publication, a contracted DuckDB mart and a Power BI model.
[Design notes](https://github.com/Kenchch/retail-ai-pipeline/blob/main/docs/DESIGN.md) ·
[Run it](https://github.com/Kenchch/retail-ai-pipeline#run-it)

## Other projects

- [nz-attraction-pageviews](https://github.com/Kenchch/nz-attraction-pageviews) — incremental Wikimedia ingestion into DuckDB with watermark recovery, quarantine and offline regression tests.
- [online-retail-analysis-r](https://github.com/Kenchch/online-retail-analysis-r) — R + SQLite analysis with a row-level cleaning audit and verification against the SHA-pinned source.
- [nz-sheep-decline-by-region](https://github.com/Kenchch/nz-sheep-decline-by-region) — Stats NZ regional livestock series with suppression-aware reconciliation; R + Quarto and a published report.
- [aerial-small-object-detection](https://github.com/Kenchch/aerial-small-object-detection) — YOLO11n on VisDrone2019, ONNX parity checks and core versus transfer-inclusive GPU/CPU latency.

Archived, kept because they still reproduce rather than because they are current: PySpark coursework implementations for [Million Song](https://github.com/Kenchch/Million-Song-Dataset-Analysis-with-Spark) and [GHCN-Daily](https://github.com/Kenchch/GHCN-Daily-Climate-Analysis-with-PySpark). Each says so at the top, with the reason its Spark pin will not move.

## How I use AI tools

I set the problem, design the data contracts and quality rules, decide what is
worth measuring, run every benchmark on my own hardware, and review every diff.
Claude Code and OpenAI Codex draft code, refactor and scaffold tests against
that. Where a finding in these repositories contradicted something I had
written, the write-up was corrected rather than the finding dropped.

Commits across these repositories carried `Co-Authored-By` trailers naming
those tools until 6 September 2026, when I rewrote the history and the trailers
went with it. That was the wrong call: it removed the disclosure without
removing the fact, and the pre-rewrite commits are still reachable on GitHub by
SHA. This section is the disclosure, kept somewhere it cannot quietly go
missing again.

[![retail CI](https://github.com/Kenchch/retail-ai-pipeline/actions/workflows/ci.yml/badge.svg)](https://github.com/Kenchch/retail-ai-pipeline/actions/workflows/ci.yml)
[![pageviews CI](https://github.com/Kenchch/nz-attraction-pageviews/actions/workflows/ci.yml/badge.svg)](https://github.com/Kenchch/nz-attraction-pageviews/actions/workflows/ci.yml)
[![R CI](https://github.com/Kenchch/online-retail-analysis-r/actions/workflows/ci.yml/badge.svg)](https://github.com/Kenchch/online-retail-analysis-r/actions/workflows/ci.yml)
[![Reproduce analysis](https://github.com/Kenchch/nz-sheep-decline-by-region/actions/workflows/reproduce.yml/badge.svg)](https://github.com/Kenchch/nz-sheep-decline-by-region/actions/workflows/reproduce.yml)
