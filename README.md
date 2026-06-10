# Ride-Hailing Data Pipeline — Thailand Market

An end-to-end data pipeline simulation for a ride-hailing platform operating across 77 Thai provinces. Built to demonstrate industry-level data engineering skills using Azure, Databricks, and Power BI.

---

## Architecture Overview

```
Data Generator
      │
      ├── Historical CSV/JSON ──► Azure Data Lake Storage (ADLS)
      │                                     │
      └── Real-time stream ──► Azure Event Hub
                                            │
                                    Databricks Pipeline
                                            │
                              ┌─────────────────────────┐
                              │   Bronze Layer           │
                              │   Raw ingestion          │
                              │   (historical + events)  │
                              └────────────┬────────────┘
                                           │
                              ┌────────────▼────────────┐
                              │   Silver Layer           │
                              │   Cleaned + enriched     │
                              │   PII hashed             │
                              └────────────┬────────────┘
                                           │
                              ┌────────────▼────────────┐
                              │   Gold Layer             │
                              │   Star schema            │
                              │   Ready for reporting    │
                              └────────────┬────────────┘
                                           │
                                      Power BI Dashboard
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Data Generation | Python (Faker, Geopy, Nominatim) |
| Real-time Ingestion | Azure Event Hub |
| Cloud Storage | Azure Data Lake Storage Gen2 |
| Pipeline Orchestration | Databricks Delta Live Tables |
| Data Processing | Apache Spark / PySpark |
| Table Format | Delta Lake |
| Data Governance | Databricks Unity Catalog |
| Reporting | Power BI |

---

## Pipeline Layers

### Bronze — Raw Ingestion
- `ingest_events.py` — reads real-time ride records from Azure Event Hub
- `ingest_historical.py` — loads historical CSV files from ADLS with manifest-based deduplication
- `ingest_mapping.py` — loads mapping tables (provinces, ride options, payment methods, etc.)

### Silver — Enrichment
- `rides_enriched.py` — combines real-time and historical records, casts timestamps, hashes PII (names, emails, phone numbers, license plates)

### Gold — Star Schema
- `star_schema.py` — builds the full star schema using Delta Live Tables

**Dimension tables:**
| Table | SCD Type | Description |
|---|---|---|
| dim_booker | Type 1 | One row per unique customer |
| dim_driver | Type 1 | One row per unique driver |
| dim_vehicle | Type 1 | One row per unique vehicle |
| dim_province | Static | 77 Thai provinces |
| dim_ride_status | Static | Completed / Cancelled |
| dim_cancellation_reason | Static | 6 cancellation reasons |
| dim_ride_option | Type 2 | Ride options with full history |
| dim_payment_method | Type 2 | Payment methods with full history |

**Fact table:**
| Table | Description |
|---|---|
| fact_rides | One row per ride with all measures and foreign keys |

---

## Data Generator

Simulates realistic ride-hailing records across Thailand using real geographic coordinates via a local Nominatim Docker instance.

**Three modes:**

```bash
# Print records to terminal (for testing)
python data_generator.py generate --count 5

# Generate historical batch and save to file
python data_generator.py historical --count 25000 --format csv --duration 2026-01-01:2026-02-01

# Stream records to Azure Event Hub
python data_generator.py eventhub --mode stream --interval 0.5
```

**Upload historical files to ADLS:**

```bash
# Upload all files
python upload_historical.py

# Upload files within a date range
python upload_historical.py --from-date 20260101 --to-date 20260601
```

---

## Power BI Dashboard

4-page interactive dashboard connected to Databricks Gold Layer.

| Page | Business Question |
|---|---|
| Growth | Is the business heading in the right direction? |
| Cancellation & Service Quality | Why are rides failing and who is responsible? |
| Geographic Performance | Where should we invest or expand? |
| Ride Option & Revenue | Which products make money and which need attention? |

---

## Project Structure

```
ride-hailing-project/
├── azure_databricks/
│   ├── pipeline-bronze/
│   │   ├── ingest_historical.py
│   │   ├── ingest_mapping.py
│   │   └── pipeline-bronze-ingestion/
│   │       └── transformations/
│   │           └── ingest_events.py
│   ├── pipeline-silver/
│   │   └── pipeline-silver-enriched/
│   │       └── transformations/
│   │           └── rides_enriched.py
│   └── pipeline-gold/
│       └── pipeline-gold-star-schema/
│           └── transformations/
│               └── star_schema.py
├── generator/
│   ├── core.py
│   ├── geocoding.py
│   ├── config.py
│   ├── pool.py
│   ├── loader.py
│   └── modes/
│       ├── generate.py
│       ├── historical.py
│       └── eventhub.py
├── settings/
│   └── storage.py
├── data/
│   ├── mapping_data/
│   ├── historical_data/
│   └── pools/
├── powerbi/
│   └── powerbi-dashboard.pbix
├── data_generator.py
├── upload_historical.py
├── generate_pools.py
├── .env.example
├── REFERENCES.md
└── requirements.txt
```

---

## Setup

**1. Clone the repository**
```bash
git clone <repo-url>
cd ride-hailing-project
```

**2. Create a virtual environment**
```bash
python -m venv .venv
.venv\Scripts\activate
```

**3. Install dependencies**
```bash
pip install -r requirements.txt
```

**4. Set up environment variables**
```bash
cp .env.example .env
# Fill in your Azure credentials in .env
```

**5. Start Nominatim Docker (for geocoding)**
```bash
docker run -p 8080:8080 mediagis/nominatim
```

---

## References

See [REFERENCES.md](REFERENCES.md) for all external documentation and resources used in this project.
