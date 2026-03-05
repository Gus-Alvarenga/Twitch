# External Data Pipeline -> Warehouse -> BI (Example Project)

This repository demonstrates an end-to-end analytics workflow: ingesting external API data, landing it in a cloud warehouse, transforming it into curated metrics, and exposing it to BI reporting.
(Example source used here: Twitch streams API.)

**TL;DR:** Twitch provides a snapshot of all current live streams, including stream metadata and current viewer counts, and that's it.  
Using this limited data, we can derive metrics as granular as how many hours a game was played, how it ranked among others, or how well a streamer is performing compared to their peers.

### Raw Data
![Twitch](https://github.com/gustavo-alvarenga/About-me/blob/main/Twitch%20Streams.png)

### BI Dashboard (Example) (Click to view the interactive [dashboard](https://app.powerbi.com/view?r=eyJrIjoiZWI2Y2M2MTgtMDYzZS00ZjBlLTlhMzAtNmJiZmVhMzdjMTBmIiwidCI6ImI1NzZhZTMzLWM3MzAtNDk5Ny1iZWY3LTQxODkxMjQzZGJkZSJ9))
![Twitch](https://github.com/gustavo-alvarenga/About-me/blob/main/Twitch%20Streams%20Dashboard.png)

### Tech Stack:
* Python
* Google Cloud Platform  
  * Compute Engine  
  * BigQuery  
  * Secret Manager
* Microsoft Fabric  
  * Dataflows  
  * Power BI
* Power Platform  
  * Power Automate
 
## Architecture (High Level)

**Flow**
1. **Ingest (VM / Compute Engine):** Poll external API on an interval and write raw results to a **BigQuery landing table** (partitioned).
2. **Transform (batch):** Aggregate raw events into curated, query-efficient metrics tables.
3. **Serve (BI):** Refresh BI dataset via Fabric Dataflows / Power BI on a schedule.
   
API -> VM ingest -> BigQuery landing -> batch transform -> BigQuery curated -> Dataflows -> Power BI

**Key design points**
- Partitioned landing table for cost control and efficient pruning.
- Batch aggregation to reduce storage and query overhead.
- Separation of concerns: ingest vs transform vs BI refresh.

## Repository Contents

- `ingest_streams.py` - Ingests raw stream snapshots from the API and appends to the BigQuery landing table.
- `process_aggregate.py` - Aggregates raw snapshots into curated metrics tables and deletes processed raw partitions (optional).
- `README.md` - Project overview and setup instructions.

## Quickstart (Reproducible Setup)

> This project is designed to run on a GCP VM. Local execution is possible if you have GCP credentials and permissions configured.

1. Create a virtual environment
   - `python -m venv .venv`
   - `source .venv/bin/activate` (Linux/Mac) or `.venv\Scripts\activate` (Windows)

2. Install dependencies
   - `pip install -r requirements.txt`

3. Configure secrets (recommended)
   - Store Twitch Client ID/Secret in GCP Secret Manager and grant the VM/service account access.
   - Update `ingest_streams.py` secret names / project settings as needed.

4. Run ingestion (VM or local)
   - `python ingest_streams.py`

5. Run processing/aggregation (batch)
   - `python process_aggregate.py`
  
## Required Configuration

This project expects:
- A Twitch application (Client ID + Client Secret)
- A GCP project with BigQuery dataset/table access
- Secrets stored in **GCP Secret Manager** (or equivalent)

At minimum, you will need to set:
- `GCP_PROJECT_ID`
- `BQ_DATASET`
- `BQ_TABLE`
- Secret names/IDs for Twitch credentials (as referenced in `ingest_streams.py`)

## Step #1: Retrieve Streams from Twitch

First, we need to retrieve stream data from Twitch. If you're unfamiliar with the [Get Streams](https://dev.twitch.tv/docs/api/reference/#get-streams) endpoint, it returns a list of all live streams at the time of the request.

To achieve granular tracking, you'll need a virtual machine running continuously. For better coverage, you can deploy multiple instances to reduce the interval between data points.

Set up a virtual machine in GCP. I won't go into full detail here, but you'll need to:
* Update and upgrade the system packages
* Install Python in a virtual environment
* Install the required libraries

Optional, but recommended:
* Set up a log file and direct script output to it
* Create a service file to manage restarts, limit memory usage, etc.

In this setup, raw snapshots are stored in BigQuery (landing table) to be processed later.

Here’s the [code](https://github.com/Gus-Alvarenga/Twitch/blob/main/ingest_streams.py).  
The script does the following:
* Retrieves data from Twitch and inserts it into a landing table (raw snapshots) BigQuery table continuously
* Manages credential refresh and standardizes data format for consistency

Assumptions:
* You have a Twitch application (Client ID and Secret)
* You're using GCP Secret Manager to store secrets (adjust accordingly if not)
* You're storing data in GCP BigQuery
* You've already created a raw snapshots table at `PROJECT_ID.twitch_temp.livestreams`, partitioned by `started_at`. Partitioning by `started_at` is strongly recommended. If you skip it, you'll need to update the query logic in the next step

The above code runs in the virtual machine you just created


## Step #2: Process Data

The raw snapshots table can grow quickly, so aggregation and lifecycle cleanup are important. You can aggregate by game, streamer, or both. For simplicity, we'll focus on aggregation by game.

Since raw data can be extremely granular, the goal is to retain detail without incurring high storage or query costs.

In this example, I run processing as a separate batch job. In production, this transformation can run either as a scheduled VM job or via warehouse-native scheduling, depending on cost, governance, and orchestration preferences.

After setting up a second VM (same steps as above), run this [code](https://github.com/Gus-Alvarenga/Twitch/blob/main/process_aggregate.py).

Process Workflow:
* Query data from the raw snapshots table
* Process and aggregate it as described
* Upload results to a permanent table
* Delete processed partitions from the raw snapshots table to keep the demo pipeline simple and storage-efficient (avoids watermark/state tracking); in production, retain raw data and use watermark-based incremental processing instead.

## Step #3: Data Visualization

Here, I use Microsoft Fabric Dataflows to pull data from BigQuery on a daily schedule. This allows for greater flexibility when managing and modifying Power BI models that rely on the same dataset.

I won't dive into the full setup, but to create the dataflow, log into the Power BI Service and configure it to fit your needs.

You can also use Power Automate to build [automated cloud flows](https://make.powerautomate.com/) for better control over refresh schedules and dataflow triggers.

Finally, here's an example [dashboard](https://app.powerbi.com/view?r=eyJrIjoiZWI2Y2M2MTgtMDYzZS00ZjBlLTlhMzAtNmJiZmVhMzdjMTBmIiwidCI6ImI1NzZhZTMzLWM3MzAtNDk5Ny1iZWY3LTQxODkxMjQzZGJkZSJ9) that can be built from this dataset:  
![Twitch](https://github.com/gustavo-alvarenga/About-me/blob/main/Twitch%20Streams%20Dashboard.png)

You can reach out to me on [LinkedIn](https://www.linkedin.com/in/gus-alvarenga/)
