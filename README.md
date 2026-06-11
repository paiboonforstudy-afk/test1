# 🚕 End-to-End Ride-Hailing Data Pipeline

## 🔎 Overview

### Project Workflow
![Overview Diagram](docs/images/overview.png)

### Dashboard Example
![Dashboard Example](docs/images/dashboard_ride_options.png)

---

## 💡 Why I Built This

I have been interested in data engineering for some time. In my previous work, I often extracted, validated, cleaned, and loaded data manually, repeating the same process whenever new data became available. While this approach was effective for smaller workloads, it was time-consuming, difficult to maintain, and not easily scalable.

To better understand how modern data platforms automate these processes, I wanted to gain hands-on experience with the tools, architectures, and workflows used in production environments. As a result, I built this project to design and implement an end-to-end data pipeline that moves raw data through ingestion, transformation, and visualization, delivering insights through an interactive dashboard.

I designed this project around two real-world pipeline scenarios:

**Scenario 1 - Real-time ingestion:** Rides data are streamed live to Azure Event Hubs, simulating a production system where data must be captured and processed continuously as it arrives.

**Scenario 2 - System migration:** The business has historical ride data from a previous system that must be migrated into the new pipeline.

---

## 📋 Table of Contents

---

## 🔄 Data Pipeline Architecture

![Data Pipeline Architecture](docs/images/pipeline_architecture.png)

The data architecture for this project follows the Medallion Architecture with Bronze, Silver, and Gold layers.

### 🥉 Bronze - Raw Ingestion

- **`ingest_events.py`** connects to Azure Event Hubs via Kafka and stores each message as a raw JSON string in `eh_rides`, parsing happens downstream in Silver.
- **`ingest_historical.py`** creates a `historical_rides_manifest` table to track every loaded file by path and file size, so only new or replaced files are ever loaded.
- **`ingest_mapping.py`** detects changes by hashing each row's content with SHA-256 and comparing it against the last known version in the target table so only new or changed rows are appended.

### 🥈 Silver - Enrichment & Privacy

- **`rides_enriched.py`** merges `eh_rides` and `historical_rides` into a single table, casts timestamp strings to proper `TIMESTAMP` types, and replaces 7 personal identification information(PII) fields (names, emails, phones, license numbers) with SHA-256 hashes.

> There is no data cleansing in this layer because the data generator always produces clean, well-formed records. In a real-world pipeline this is where null handling, deduplication, and format validation would live.

### 🥇 Gold - Star Schema

![Star Schema](docs/images/star_schema.png)

- **`star_schema.py`** builds the full star schema from `rides_enriched` and the bronze mapping tables.
- **`dim_booker`, `dim_driver`, `dim_vehicle`** are SCD Type 1 - always reflects the latest value, no history kept.
- **`dim_ride_option`, `dim_payment_method`** are SCD Type 2 - full history is kept so old rides always link to the correct version at the time of booking.
- **`dim_province`, `dim_ride_status`, `dim_cancellation_reason`** are static reference tables loaded directly from bronze.
- **`fact_rides`** stores one row per ride with all foreign keys, timestamps, fare breakdown, distance, duration, and ratings.

---

## 📊 Dashboard

4-page interactive dashboard built on the Gold layer star schema.

| Page | Business Question |
|---|---|
| 📈 **Growth** | Is the business growing sustainably and moving in the right direction?|
| ❌ **Cancellation & Service Quality** | What factors are driving ride cancellation, and where are service improvements needed? |
| 🗺️ **Geographic Performance** | Which regions are performing best, and where should future investments or expansion be focused? |
| 💰 **Ride Option & Revenue** | Which ride option generate the highest revenue, and which require strategic attention? |

> Note: The dashboard data is generated for demonstration purposes and does not represent real-world figures, which is why some visuals may not make sense.

### 📈 Growth
![Dashboard Growth](docs/images/dashboard_growth.png)

### ❌ Cancellation & Service Quality
![Dashboard Cancellation & Service Quality](docs/images/dashboard_cancellation.png)

### 🗺️ Geographic Performance
![Dashboard Geographic Performance](docs/images/dashboard_geographic_performance.png)

### 💰 Ride Option & Revenue
![Dashboard Ride Options](docs/images/dashboard_ride_options.png)

---

## ⚙️ Data Generator

![Data Generator Diagram](docs/images/data_generator.png)

**Step 1 — Generate pools (`generate_pools.py`)**

Before any rides can be generated, a pool of drivers and customers must be created. `generate_pools.py` pre-generates a configurable number of unique drivers and customers and saves them to files. The generator reuses these pools across runs so that the same people appear in multiple rides, simulating real repeat users and drivers.

**Step 2 — Generate rides (`data_generator.py`)**

Once the pools are ready, `data_generator.py` generates ride records using real Thai geographic coordinates validated against a self-hosted Nominatim (OpenStreetMap) instance. Two modes are used:

- **`eventhub` mode** - streams live ride records one by one to Azure Event Hubs as JSON. Simulates real-time ride records.
- **`historical` mode** - generates a batch of rides within a given date range and saves them as CSV or JSON files locally. Used to generate historical data.

After historical files are generated, `upload_historical.py` uploads them to Azure Data Lake Storage Gen2, where they are picked up by the Bronze ingestion pipeline.

**Mapping data - GitHub Actions**

Province, ride option, and payment method reference files are stored in the repository under `data/mapping_data/`. A GitHub Actions workflow automatically uploads these JSON files to Azure Data Lake Storage Gen2 whenever they are updated in the repository.

### Commands : <link>
### How location generation works : <link>

---

## 📐 Data Structure

**Dimension tables:**

| Table | SCD Type | Key Columns |
|---|---|---|
| `dim_booker` | Type 1 | booker_id |
| `dim_driver` | Type 1 | driver_id |
| `dim_vehicle` | Type 1 | vehicle_id |
| `dim_province` | Static | province_id, province_name |
| `dim_ride_status` | Static | ride_status_id, ride_status |
| `dim_cancellation_reason` | Static | cancellation_reason_id, initiator, cancellation_reason |
| `dim_ride_option` | Type 2 | ride_option_id, ride_option_name, base_rate, per_km, per_minute |
| `dim_payment_method` | Type 2 | payment_method_id, payment_method, is_card |

**Fact table:**

| Table | Grain | Key Measures |
|---|---|---|
| `fact_rides` | One row per ride | total_fare, travel_distance_km, duration_minutes, surge_multiplier, tip_amount, rating, driver_rating |

> **SCD Type 1** — always reflects the latest value, no history kept.
> **SCD Type 2** — keeps a full history of changes. Old rides always link to the correct version of the ride option or payment method at the time of booking.

---

## 🗂️ Project Structure

```
ride-hailing-project/
├── azure_databricks/              # Databricks pipeline notebooks
│   ├── pipeline-bronze/
│   │   ├── ingest_historical.py
│   │   ├── ingest_mapping.py
│   │   └── pipeline-bronze-ingestion/transformations/
│   │       └── ingest_events.py
│   ├── pipeline-silver/
│   │   └── pipeline-silver-enriched/transformations/
│   │       └── rides_enriched.py
│   └── pipeline-gold/
│       └── pipeline-gold-star-schema/transformations/
│           └── star_schema.py
├── generator/                     # Data generation logic
│   ├── core.py                    # Ride record simulation
│   ├── geocoding.py               # Nominatim location generator
│   ├── config.py                  # Probabilities and parameters
│   ├── pool.py                    # Driver and customer pools
│   ├── loader.py                  # Mapping data loader
│   └── modes/
│       ├── generate.py            # Terminal output mode
│       ├── historical.py          # File output mode
│       └── eventhub.py            # Azure Event Hub mode
├── settings/
│   └── storage.py                 # Paths and Azure settings
├── data/
│   ├── mapping_data/              # Province, ride option, payment method JSON
│   ├── historical_data/           # Generated CSV/JSON ride files
│   └── pools/                     # Driver and customer pool files
├── powerbi/
│   └── powerbi-dashboard.pbix    # Power BI dashboard
├── docs/
│   └── images/                   # Architecture and dashboard screenshots
├── data_generator.py              # CLI entry point
├── upload_historical.py           # Upload files to ADLS
├── generate_pools.py              # Pre-generate driver/customer pools
├── .env.example                   # Required environment variables
├── REFERENCES.md                  # External documentation
└── requirements.txt
```

---

## 🚀 Setup

**1. Clone the repository**
```bash
git clone <repo-url>
cd ride-hailing-project
```

**2. Create virtual environment**
```bash
python -m venv .venv
.venv\Scripts\activate       # Windows
source .venv/bin/activate    # Mac / Linux
```

**3. Install dependencies**
```bash
pip install -r requirements.txt
```

**4. Set up environment variables**
```bash
cp .env.example .env
# Fill in your Azure credentials
```

**5. Start Nominatim Docker**
```bash
docker run -p 8080:8080 mediagis/nominatim
```

**6. Generate data**
```bash
python data_generator.py historical --count 5000 --format csv --duration 2026-01-01:2026-02-01
```

**7. Upload to Azure**
```bash
python upload_historical.py
```

---

## 📚 References

See [REFERENCES.md](REFERENCES.md) for all external documentation and resources used in this project.
