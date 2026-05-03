# 🌩️ Weather-Based Delivery Disruption Predictor

A batch data pipeline and machine learning system that predicts whether weather conditions are likely to cause disruptions in delivery services. The system ingests real-time weather data every 15 minutes, stores it in an object store, and applies ML classification models built with Spark MLlib to flag high-risk delivery windows.

---

## 📋 Table of Contents

- [Objective](#objective)
- [Architecture Overview](#architecture-overview)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Data Pipeline](#data-pipeline)
  - [Ingestion](#ingestion)
  - [Storage](#storage)
  - [Processing & Modeling](#processing--modeling)
- [ML Models & Results](#ml-models--results)
- [Limitations & Future Work](#limitations--future-work)
- [Getting Started](#getting-started)

---

## Objective

Delivery operations are highly sensitive to weather conditions. Rain, strong winds, or extreme temperatures can cause delays, accidents, or service cancellations. This project builds an automated pipeline that:

1. Fetches weather data every 15 minutes from the OpenMeteo API
2. Stores raw data in a MinIO object store
3. Applies a binary classification model (disruption / no disruption) trained with Spark MLlib
4. Compares multiple ML algorithms to identify the best-performing approach

The output is a prediction label per 15-minute interval that can be consumed by downstream systems such as dashboards, alerting tools, or scheduling systems.

---

## Architecture Overview

```
┌─────────────────┐     every 15 min     ┌───────────────────┐
│  OpenMeteo API  │ ──────────────────► │   Apache NiFi      │
│  (Weather Data) │                      │  (Ingestion Layer) │
└─────────────────┘                      └────────┬──────────┘
                                                   │
                                                   ▼
                                         ┌───────────────────┐
                                         │   MinIO           │
                                         │  (Object Store)   │
                                         │  Raw / Processed  │
                                         └────────┬──────────┘
                                                   │
                                                   ▼
                                         ┌───────────────────┐
                                         │   Apache Spark    │
                                         │   MLlib           │
                                         │  (ML Pipeline)    │
                                         └────────┬──────────┘
                                                   │
                                                   ▼
                                         ┌───────────────────┐
                                         │  Prediction Output│
                                         │  Disruption: Y/N  │
                                         └───────────────────┘
```

---

## Tech Stack

| Layer | Tool | Purpose |
|---|---|---|
| **Ingestion** | Apache NiFi | Scheduled API polling, data routing |
| **Storage** | MinIO | S3-compatible object store for raw & processed data |
| **Processing** | Apache Spark (PySpark) | Distributed data processing and ML |
| **ML** | Spark MLlib | Model training, evaluation and comparison |
| **Data Source** | OpenMeteo API | Free, open-source weather forecast API |


---

## Data Pipeline

### Ingestion

Apache NiFi handles the ingestion layer using a scheduled flow that:

- Calls the [OpenMeteo API](https://open-meteo.com/) every **15 minutes** via an `InvokeHTTP` processor
- Fetches the following weather variables for a configured latitude/longitude:
  - `temperature_2m` — Air temperature at 2m (°C)
  - `relative_humidity_2m` — Relative humidity (%)
  - `windspeed_10m` — Wind speed at 10m (km/h)
  - `windgusts_10m` — Wind gusts at 10m (km/h)
  - `precipitation` — Precipitation (mm)
  - `weathercode` — WMO weather condition code
  - `visibility` — Visibility (m)
- Adds an ingestion timestamp and routes the JSON payload to MinIO via a `PutS3Object` processor configured to point to the local MinIO endpoint

The NiFi flow template can be imported from `nifi/flows/weather_ingestion.xml`.

### Storage

MinIO provides an S3-compatible object storage layer with two buckets:

| Bucket | Content |
|---|---|
| `raw-weather` | Raw JSON responses from the API, partitioned by `year/month/day/hour` |

### Processing & Modeling

The Spark pipeline reads from MinIO, engineers features, generates disruption labels, and trains multiple classifiers.

**Feature Engineering** (`spark/preprocessing.py`):

- Parses raw JSON and casts all fields to the correct types
- Derives a `disruption` binary label using deterministic weather thresholds (see Limitations section)
- Adds time-based features: `hour_of_day`, `day_of_week`, `is_weekend`
---

## ML Models

| Model | Notes |
|---|---|
| Logistic Regression | Baseline linear classifier |
| Gradient Boosted Trees | Boosting approach, sensitive to class imbalance |

Models are evaluated using **AUC-ROC**, **F1-score**, and **accuracy** on a held-out test set.

> ⚠️ The dataset used for training reflects labeled outputs from the deterministic rule engine, not real disruption observations. See the Limitations section for important context.

---

## Limitations & Future Work

### Label Quality

The disruption label is generated using **deterministic thresholds** applied to raw weather variables (e.g., wind speed > 50 km/h, precipitation > 10 mm). While this provides a reasonable proxy, it does not reflect actual observed disruptions. A significantly better approach would be to use **historical disruption records** from a logistics or courier system and join them with the weather data to train on real-world outcomes.

### Update Frequency

The pipeline fetches data every 15 minutes, which introduces an inherent latency in predictions. For operational use cases where disruption alerts need to be issued in advance, a **real-time streaming approach** would be more suitable — for example, using a data source that provides minute-level or sub-minute weather updates and processing it through a streaming framework like Spark Structured Streaming or Kafka Streams.

### Geographic Scope

Currently the pipeline is configured for a single location (latitude/longitude in `config/settings.py`). Extending it to multiple delivery regions would require parameterising the NiFi flow and partitioning the storage and model outputs by region.

### Model Serving

Predictions are currently produced as a batch job. A future iteration could expose predictions via a lightweight REST API so that downstream systems can query the disruption risk for any upcoming time window on demand.

---




## License

MIT License. See `LICENSE` for details.
