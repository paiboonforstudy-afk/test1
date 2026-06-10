# End-to-End Ride-Hailing Data Pipeline — Thailand Market

A production-style data pipeline that simulates a ride-hailing platform operating across all 77 Thai provinces. Built to demonstrate real-world data engineering — from data generation to cloud ingestion, transformation, and business intelligence reporting.

> **Scale:** 30,000+ ride records · 77 provinces · 6 ride options · 5 payment methods

---

## Architecture

![Architecture Diagram](docs/images/architecture.png)

```
┌─────────────────────────────────────────────────────────────────┐
│                        Data Generator                           │
│   Python · Faker · Nominatim · Real Thai Coordinates            │
└──────────────────┬──────────────────────┬───────────────────────┘
                   │                      │
         Historical Batch            Real-time Stream
         CSV / JSON files            Azure Event Hub
                   │                      │
                   └──────────┬───────────┘
                              │
             ┌────────────────▼─────────────────┐
             │   Azure Data Lake Storage Gen2   │
             └────────────────┬─────────────────┘
                              │
             ┌────────────────▼─────────────────┐
             │         Databricks               │
             │                                  │
             │  Bronze ──► Silver ──► Gold      │
             │  Raw       Cleaned    Star Schema │
             └────────────────┬─────────────────┘
                              │
             ┌────────────────▼─────────────────┐
             │         Power BI Dashboard       │
             │  Growth · Cancellation ·         │
             │  Geographic · Ride Option        │
             └──────────────────────────────────┘
```

---

## Dashboard Preview

![Power BI Dashboard](docs/images/dashboard_overview.png)

| Page | Business Question |
|---|---|
| Growth | Is the business heading in the right direction? |
| Cancellation & Service Quality | Why are rides failing and who is responsible? |
| Geographic Performance | Where should we invest or expand? |
| Ride Option & Revenue | Which products make money and which need attention? |

---

## Data Generator

The generator creates realistic ride records using **real geographic coordinates** across Thailand — not random lat/lon values. It uses a self-hosted **Nominatim** instance (OpenStreetMap) running in Docker to reverse-geocode coordinates and validate that each point is on land, inside Thailand, and within the correct province.

### How location generation works

```
1. Request bounding box for a province from Nominatim
         e.g. "Bangkok, Thailand" → lat/lon bounds

2. Sample a random coordinate within the bounding box

3. Reverse geocode the coordinate
         → check: is it on land?
         → check: is it inside Thailand?
         → check: does province_id match?

4. If all checks pass → use this coordinate
   If any check fails → retry (up to 500 attempts)
```

This ensures every pickup and dropoff location in the dataset is a real, valid location inside the correct Thai province.

### Realistic simulation parameters

| Parameter | Value | Detail |
|---|---|---|
| Hot provinces | 11 | Bangkok, Phuket, Chiang Mai, Pattaya, etc. |
| Hot province selection chance | 80% | Simulates real urban demand concentration |
| Same-province dropoff chance | 70% | Most rides stay within one province |
| Completion rate | 80% | 20% of rides are cancelled |
| Surge hours | 7–9 AM, 5–8 PM | Surge multiplier 1.2x–1.5x |
| Tip chance | 20% | 5–20% of subtotal |
| Driver pool | 500 drivers | Reused across rides to simulate real drivers |
| Customer pool | 5,000 customers | Reused across rides to simulate repeat users |

### Ride option distance suitability

Each ride option is only assigned to appropriate trip distances:

| Ride Option | Short (0–5km) | Medium (5–15km) | Suburban (15–80km) | Regional (80–250km) | Cross-country |
|---|---|---|---|---|---|
| Economy | ✅ | ✅ | ✅ | ❌ | ❌ |
| Taxi | ✅ | ✅ | ✅ | ❌ | ❌ |
| Bike | ✅ | ✅ | ❌ | ❌ | ❌ |
| Premium | ❌ | ✅ | ✅ | ✅ | ❌ |
| SUV | ❌ | ✅ | ✅ | ✅ | ✅ |
| Van | ❌ | ❌ | ✅ | ✅ | ✅ |

### Generator commands

```bash
# Print 5 records to terminal (for testing)
python data_generator.py generate --count 5

# Generate 25,000 historical records and save to CSV
python data_generator.py historical --count 25000 --format csv --duration 2026-01-01:2026-02-01

# Stream live records to Azure Event Hub
python data_generator.py eventhub --mode stream --interval 0.5
```

---

## Data Pipeline — Bronze → Silver → Gold

### Bronze — Raw Ingestion

| Script | Source | Target | Description |
|---|---|---|---|
| `ingest_events.py` | Azure Event Hub | `bronze.eh_rides` | Real-time ride records via Kafka |
| `ingest_historical.py` | ADLS CSV files | `bronze.historical_rides` | Batch historical rides with manifest deduplication |
| `ingest_mapping.py` | ADLS JSON files | `bronze.map_*` | Mapping tables (provinces, ride options, etc.) |

### Silver — Enrichment & Privacy

| Script | Source | Target | Transformations |
|---|---|---|---|
| `rides_enriched.py` | `bronze.eh_rides` + `bronze.historical_rides` | `silver.rides_enriched` | Cast timestamps · Hash PII (SHA-256) |

**PII fields hashed:** `booker_name`, `booker_email`, `booker_phone`, `driver_name`, `driver_phone`, `driver_license`, `vehicle_license_plate`

### Gold — Star Schema

![Star Schema](docs/images/star_schema.png)

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

> **SCD Type 1** — always reflects the latest known value, no history kept.
> **SCD Type 2** — keeps a full history of changes over time. Old rides always link back to the correct version of the ride option or payment method at the time of the ride.

---

## Project Structure

```
ride-hailing-project/
├── azure_databricks/              # Databricks pipeline notebooks
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
├── generator/                     # Data generation logic
│   ├── core.py                    # Main ride record simulation
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
│   ├── historical_data/           # Generated CSV / JSON ride files
│   └── pools/                     # Driver and customer pool files
├── powerbi/
│   └── powerbi-dashboard.pbix    # Power BI dashboard
├── data_generator.py              # CLI entry point
├── upload_historical.py           # Upload files to ADLS
├── generate_pools.py              # Pre-generate driver/customer pools
├── .env.example                   # Required environment variables
├── REFERENCES.md                  # External documentation
└── requirements.txt
```

---

## Setup

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

## References

See [REFERENCES.md](REFERENCES.md) for all external documentation and resources used in this project.
