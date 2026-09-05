## Feng Jiang

**Data and analytics engineering — Python, SQL, Spark, Airflow and dbt.**

Based in Christchurch, New Zealand, and open to data and analytics engineering roles. [Contact](mailto:janfinq@gmail.com).

I build pipelines with explicit data contracts, recoverable publication and reconciled outputs. This portfolio also covers statistical analysis and measured ML deployment trade-offs.

## Selected engineering work

| Project | Implementation and evidence | Scope and trade-offs |
|---|---|---|
| [**retail-ai-pipeline**](https://github.com/Kenchch/retail-ai-pipeline) | Airflow orchestration, versioned publication through an atomic manifest, a contracted dbt/DuckDB mart and a Power BI semantic model. **522,566 loaded + 19,343 quarantined = 541,909 input rows**. | Single-machine storage and execution. Warehouse and downstream mart promotion are separate boundaries; adoption telemetry is simulated. |
| [**nz-attraction-pageviews**](https://github.com/Kenchch/nz-attraction-pageviews) | Incremental Wikimedia ingestion into DuckDB: retries, quarantine, watermark recovery and an [offline demo](https://github.com/Kenchch/nz-attraction-pageviews#run-it). | Handles ambiguous missing responses and publication lag; pageviews measure online attention, not attendance. See [limits](https://github.com/Kenchch/nz-attraction-pageviews#limits). |
| [**aerial-small-object-detection**](https://github.com/Kenchch/aerial-small-object-detection) | YOLO11n on VisDrone2019, ONNX export and accuracy parity checks, CUDA placement verification, and separate core/transfer-inclusive latency measurements. | Validation-based checkpoint selection and a limited training budget. Synthetic-motion tracking demo demonstrates the pipeline, not tracking accuracy on moving objects. |

## Analysis and distributed processing

| Project | Technical focus | Evidence |
|---|---|---|
| [**online-retail-analysis-r**](https://github.com/Kenchch/online-retail-analysis-r) | R/SQLite analysis with chronological, one-to-one credit matching, SHA-256 input verification and checks against committed summary tables. | [Rendered analysis](https://github.com/Kenchch/online-retail-analysis-r/blob/main/analysis.md), including the published correction and exact-match limitations. |
| [**nz-sheep-decline-by-region**](https://github.com/Kenchch/nz-sheep-decline-by-region) | Stats NZ source reconciliation, suppression-aware comparisons and sensitivity to the choice of observation window; R + Quarto. | [Published report](https://kenchch.github.io/nz-sheep-decline-by-region/), with validation rules exercised against deliberately corrupted data. |
| [**Million Song**](https://github.com/Kenchch/Million-Song-Dataset-Analysis-with-Spark) | PySpark genre models and implicit-feedback ALS; training-only resampling, ranking-metric tests and real-Spark model smoke tests. | [Recorded study results and provenance](https://github.com/Kenchch/Million-Song-Dataset-Analysis-with-Spark/blob/main/docs/results.md). Historical results are distinct from the current CLI and synthetic CI fixtures. |
| [**GHCN-Daily**](https://github.com/Kenchch/GHCN-Daily-Climate-Analysis-with-PySpark) | Fixed-width metadata parsing, station enrichment and weather aggregation in Spark; synthetic regression tests exercise quality flags and unit conversion. | [Workflow and interpretation limits](https://github.com/Kenchch/GHCN-Daily-Climate-Analysis-with-PySpark#reproducibility-notes). Station summaries are not coverage-adjusted national climate estimates; CI does not establish archive-scale performance. |

## Reading the results

These are portfolio and study projects, not evidence of production deployment or realised commercial impact. Dataset revenue totals and offline model benchmarks are reported with their respective definitions and limitations.

The two retail projects use the same UCI source but different acceptance and cancellation rules. Their READMEs reconcile gross positive sales with the R analysis's matched-cancellation result; partial and unmatched returns remain outside that netting method.

## How this portfolio was built

I used AI pair-programming tools, including Claude Code and OpenAI Codex, for drafting, refactoring and test scaffolding. I defined the problems, designed the pipelines and data contracts, selected the quality rules, ran the benchmarks on my own hardware, and reviewed and edited the resulting code. Commits with assistant-contributed code retain their `Co-Authored-By` trailers.
