## Feng Jiang

I build data pipelines and analyses that make it possible to trace a result back to its source.

Based in **Christchurch, New Zealand**, and open to **data and analytics engineering roles**. [Email me](mailto:janfinq@gmail.com).

## Explore my work

The links below lead to reports, demonstrations and source code. You can read the results without installing anything.

| Project | The question or problem | Start with |
|---|---|---|
| **Retail data pipeline** | How can sales data be checked and published reliably, even when a run fails? | [Pipeline and Power BI preview](https://github.com/Kenchch/retail-ai-pipeline) |
| **NZ attraction pageviews** | How can a daily data import recover from missing API responses without losing records? | [Small offline demo](https://github.com/Kenchch/nz-attraction-pageviews#run-it) |
| **Online retail analysis** | How do cancellations change the picture of sales and repeat customers? | [Analysis with charts](https://github.com/Kenchch/online-retail-analysis-r/blob/main/analysis.md) |
| **NZ sheep decline** | Which regions account for the decline, and what can the published statistics tell us? | [Read the published report](https://kenchch.github.io/nz-sheep-decline-by-region/) |
| **Drone object detection** | How does image resolution affect the detection of tiny objects and inference speed? | [Watch the demo and inspect results](https://github.com/Kenchch/aerial-small-object-detection) |
| **Million Song** | How can Spark support music classification and recommendations across millions of listening events? | [Pipelines and recorded study results](https://github.com/Kenchch/Million-Song-Dataset-Analysis-with-Spark) |
| **GHCN-Daily climate analysis** | How can a large weather archive become station and country summaries? | [Spark workflows and example charts](https://github.com/Kenchch/GHCN-Daily-Climate-Analysis-with-PySpark) |

For **data engineering**, start with the retail pipeline and NZ pageviews. For a **visual analysis**, start with the NZ sheep report.

## What the results mean

These are portfolio and study projects. Sales totals describe historical datasets, and model scores describe experiments; they are not revenue I generated or measured customer impact. The retail pipeline's adoption telemetry is simulated.

The two retail projects use the same source data with different cleaning and cancellation rules. Their READMEs explain and reconcile the different sales totals. Each project documents its own reproduction steps, tests and limitations.

## Tools and approach

My core tools are **Python, SQL and R**, with Spark for distributed processing, Airflow and dbt for pipelines, Power BI for reporting, and PyTorch/ONNX for applied ML.

I focus on source-data checks, recoverable workflows, automated tests and evidence that a reader can inspect.

## How this portfolio was built

I used AI pair-programming tools, including Claude Code and OpenAI Codex, for drafting, refactoring and test scaffolding. I defined the problems, designed the pipelines and data contracts, selected the quality rules, ran the benchmarks on my own hardware, and reviewed and edited the resulting code. Commits with assistant-contributed code retain their `Co-Authored-By` trailers.
