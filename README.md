# &#x20;Bee Haven: an end-to-end Azure data pipeline for beehive sensor data

An automated **Medallion Lakehouse** (Bronze → Silver → Gold) on Microsoft Azure that cleans 2.5 years of beehive sensor data,
enriches it with historical weather from the **BrightSky API**, and delivers analysis-ready tables, orchestrated nightly by **Azure Data Factory**
and processed in **Azure Synapse** notebooks.

**Business question:** *What makes bees busy?* How do temperature, humidity and weight influence bee traffic and honey production?

[Daily bee traffic, hive temperature, humidity and weight]

---

## Architecture

![Dashboard Screenshot](image/WIP_III.png)


**Orchestration (Azure Data Factory), all activities succeeded:**
`Copy` source → bronze/new → `Get Metadata` → `ForEach` (archive each file with a timestamp) → `Copy` to silver/processing
→ **Synapse Notebook 04** → `Delete` silver/processing → **Synapse Notebook 05** → `Delete` gold/processing.
A daily **schedule trigger** runs the whole chain. If a notebook's quality checks fail, the run stops and the files stay in the processing folder for inspection.

image/datafactoryed09_Activity runs(1).csv
[Azure Data Factory pipeline]

## Azure Data Lake Storage structure

The ADLS Gen2 container `newdatalakemedallion33` is organised into three medallion layers and a Synapse workspace folder:

```text
newdatalakemedallion33/                 # ADLS Gen2 container
├── bronze/
│   ├── new/                             # Incoming source files for processing
│   └── archive/                         # Timestamped copies of source files
│       ├── flow/
│       ├── humidity/
│       ├── temperature/
│       ├── weather/
│       └── weight/
├── silver/
│       ├── flow/
│       ├── humidity/
│       ├── temperature/
│       ├── weather/
│       └── weight/
│       ├── processing/                             # Cleaned and processing data
├── gold/
│   └── processing/                      # Temporary files used by the gold-processing step
└── synapse/                             # Synapse-related files, if used by the workspace
```

The exact subfolders present can vary as the pipeline runs. The pipeline copies incoming files to the archive, processes data through the silver and gold stages, and removes temporary processing files after successful notebook activities.

> **Storage note:** The container holds the data; GitHub should hold the project code, notebooks, Data Factory definitions, documentation and screenshots—not large raw or processed Parquet files.

## Azure Data Factory pipeline: implemented workflow

The `bronzenewmetadata` pipeline orchestrates the data movement and notebook processing:

1. **Copy source data** into `bronze/new`.
2. **Get Metadata** to identify the incoming files.
3. **ForEach** file: create an archive copy under `bronze/archive` with a timestamp.
4. **Copy** the selected files into `silver/processing`.
5. **Run the silver-processing Synapse notebook** to clean and prepare the data.
6. **Delete temporary silver processing files** after successful processing.
7. **Run the gold-processing Synapse notebook** to produce analysis-ready outputs.
8. **Delete temporary gold processing files** after successful processing.

The Azure Data Factory run shown in the portal completed successfully, with all displayed activities marked **Succeeded**. The pipeline is configured with a daily schedule trigger.

If a notebook activity fails, check its activity output and the files left in the processing folder before rerunning the pipeline. Avoid deleting processing files manually until you have confirmed that the run has completed and the outputs are valid.

## Azure implementation screenshots

Place the relevant portal screenshots in `images/azure/` and keep the filenames below consistent with the repository:

| Screenshot | What it documents |
| --- | --- |
| `images/azure/02_data_lake_containers.png` | Container and medallion folders |
| `images/azure/02b_bronze_archive_structure.png` | Bronze archive subfolders |
| `images/azure/03b_adf_pipeline_canvas.png` | Data Factory pipeline activities and dependencies |
| `images/azure/03_adf_pipeline_run_succeeded.png` | Successful pipeline run and activity statuses |


The full Data Factory definition (pipelines, datasets, linked services and trigger, exported as an ARM template) is in
, including an activity-by-activity description.

## Tech stack

| Area                   | Tools                                                                                                |
| ---------------------- | ---------------------------------------------------------------------------------------------------- |
| Storage                | Azure Data Lake Storage Gen2 (hierarchical namespace, medallion folders)                             |
| Orchestration          | Azure Data Factory: Get Metadata, ForEach, Copy, Notebook, Delete, schedule trigger, dynamic content |
| Processing             | Azure Synapse Analytics: Apache Spark pool running Python notebooks                                  |
| Language and libraries | Python, pandas, NumPy, fsspec / adlfs, requests, matplotlib                                          |
| File formats           | CSV (raw), JSON (API, semi-structured), **Parquet** (columnar, silver and gold)                      |
| Security               | Azure RBAC / IAM: managed identity for Data Factory, Synapse RBAC roles                              |
| Infrastructure as code | ARM template export of the whole Data Factory                                                        |

## The data

| Source                           | Frequency | Size      | Notes                                            |
| -------------------------------- | --------- | --------- | ------------------------------------------------ |
| Bee flow (departures / arrivals) | 1 min     | 2.5M rows | Two series stacked in one file                   |
| Hive temperature                 | 5 min     | 253k rows |                                                  |
| Hive humidity                    | 12 h      | 1.8k rows |                                                  |
| Hive weight                      | 12 h      | 1.8k rows | grams                                            |
| Weather (BrightSky / DWD)        | 1 h       | 21k rows  | Temperature, rain, sunshine, wind, cloud cover … |

Period: January 2017 to May 2019, hive in Bad Schwartau and in pipeline added wurzburg Germany also.

## Data-quality issues found and how they were solved

Every decision is backed by a chart in `01_Data_prep_sample_solution_annotated.ipynb`.

| Issue found                                                                | Decision                            | Why                                                                                                                         |
| -------------------------------------------------------------------------- | ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Timestamps stored as text, in German local time                            | Converted to **UTC**                | Daylight saving makes 02:00–02:59 happen twice in October; UTC gives every moment one timestamp and matches the weather API |
| Flow file = departures stacked on top of arrivals                          | Split into `flow_out` / `flow_in`   | One timestamp, one row, clear meaning                                                                                       |
| 697k "duplicate" rows in raw flow                                          | **Kept**                            | They are real zero counts (night) appearing once in each half; dropping them would delete data                              |
| Humidity of −100 % next to 100 %                                           | Absolute value                      | Sign-flip sensor error                                                                                                      |
| Weight of −170 g after the hive was emptied                                | Clipped to 0                        | A scale can't weigh less than nothing                                                                                       |
| Arrivals of 7,999 bees/min, and spikes at night                            | Set to NaN                          | Error code; more arrivals than the busiest departure minute ever is physically impossible                                   |
| IQR / z-score outlier rules                                                | Tested, **not used**                | Bee traffic is extremely skewed; they would flag \~175k normal busy minutes                                                 |
| Missing values and sensor outages                                          | Kept as NaN                         | Silver doesn't invent data; filling gaps is an analysis decision                                                            |
| Weather values borrowed from more distant stations (`fallback_source_ids`) | `<measure>_source_distance` columns | Analysts can see how far away each value was measured                                                                       |



## Gold layer: analysis-ready tables

| Table                     | Grain                                 | Content                                                                                            |
| ------------------------- | ------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `hivedata_hourly` ★       | 1 row per hour per hive               | Departures, arrivals, hive temperature/humidity/weight + all weather measures, aligned on UTC hour |
| `hivedata_daily`          | 1 row per day                         | Daily totals (bees, rain, sunshine hours) and averages, plus **daily weight change**               |
| `activity_by_time_of_day` | morning / afternoon / evening / night | Average bees per hour                                                                              |
| `activity_by_temperature` | 5 °C outside-temperature bands        | Daytime bees per hour                                                                              |

Why **hourly**: the sensors tick at different rates (1 min, 5 min, 12 h) and weather is hourly. Joining on exact timestamps leaves
most cells empty, so aggregating to hours gives every row its matching weather. Counts are **summed**, measurements **averaged**.

- Bee traffic rises steeply once the hive is above \~20 °C (correlation +0.66 with hive temperature).
- The hive gets **drier on busy days** (−0.74), likely because bees fan their wings to dry nectar into honey.
- Weight is a running total, so the **daily change** in weight, not its level, is what relates to activity.
- The colony keeps the brood nest at \~35 °C all summer, regardless of the weather outside.

## Repository structure

```
├── notebooks/
│   ├── 01_bronze_to_silver_local.ipynb        # exploration + cleaning decisions, with charts
│   ├── 02_weather_sample_solution.ipynb         # BrightSky API → bronze JSON → silver Parquet
│   ├── 03_silver_processing_to_silver_and_to_gold_processing_KE_new.ipynb   # pipeline notebook (Synapse)
│   └── 05_gold_processing_to_gold_KE_new.ipynb       # pipeline notebook (Synapse): gold tables
├── azure-data-factory/
│   ├── pipeline_BronzeNew_To_BronzeArchive.json   # main pipeline
│   ├── pipeline_Lake_old_to_broze_new.json
├── images/                                    # charts + images/azure/ screenshots
└── README.md


Built as part of the Data analytics & AI programme at WBS Coding School, aligned with the **Microsoft DP-900 (Azure Data Fundamentals)** certification.
