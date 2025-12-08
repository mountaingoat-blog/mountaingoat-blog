---
layout: post
title: "Gold Prediction – Continuous Ingestion & Curation Platform Architecture"
date: 2025-12-08 12:00:00 +0530
categories: data-platform gold
tags: [gold, ingestion, knime, aws, eventbridge, s3]
---

This architecture powers the continuous ingestion, syncing, and automated curation of all datasets required for Gold Prediction at Mountaingoat.  
It integrates Python-based data fetchers, Lightsail Object Storage, a central AWS S3 datalake, EventBridge-based event routing, and KNIME batch workflow for automated dataset curation.

---

## 1. Data Fetching Layer (Lightsail VM)

A Lightsail VM runs Python fetchers that pull gold and macroeconomic data from external APIs and store them in Lightsail Object Storage.

### Gold Fetchers

**fetch_one_day_ohlc_parquet.py**  
Fetches XAU/USD daily OHLC from TwelveData and writes a single-row Parquet file.

**fetch_one_hour_ohlc_parquet.py**  
Fetches XAU/USD hourly OHLC and writes a single-row Parquet file.

### FRED Fetcher

**pull_fred_series_to_s3_csv_dedupe.py**  
Downloads macroeconomic series (DFF, DFII10, DGS10, DGS2, DTWEXBGS, VIXCLS, DCOILWTICO) and uploads CSV only if the latest observation has changed.

All outputs are stored in Lightsail Object Storage (S3-compatible).

---

## 2. Sync Layer — Lightsail → AWS S3

A sync script periodically copies all new raw files from Lightsail to the AWS S3 bucket `gold-prediction-01`.

### Sync Script

    lightsail_sync_full.sh

### Sync Flow

1. Lightsail bucket → local temporary directory  
2. Local temporary directory → AWS S3 (gold-prediction-01)

AWS S3 becomes the single source of truth for all raw data.

---

## 3. Event Routing via EventBridge

EventBridge monitors S3 prefixes and forwards object-created events to SQS.

### Monitored Prefixes

- GOLD_OHLC_Daily_Rate_USD/  
- GOLD_OHLC_Live_Rate_USD/  
- us-macro-economic-data/DFF/  
- us-macro-economic-data/VIXCLS/  

### Destination Queue

**knime-gold-events**

Every new file triggers an SQS event.

---

## 4. KNIME Poller on the KNIME Server

A custom poller continuously listens to SQS and triggers the matching KNIME workflow.

### Poller Path

    /mnt/knime-data/poller/bin/knime-gold-poller-single.sh

### Poller Responsibilities

- Long-poll SQS  
- Extract S3 object key from event  
- Map prefix to KNIME workflow  
- Create per-run temp directory  
- Invoke KNIME in batch mode  

### KNIME Batch Invocation

    knime --launcher.suppressErrors -nosplash -nosave \
      -application org.knime.product.KNIME_BATCH_APPLICATION \
      -workflowDir="<workflow>" \
      -reset -consoleLog -Xmx6G

### Message Handling

- On success → delete message  
- On failure → message stays for retry  
- Per-run logs stored for debugging  

This forms a fully event-driven ETL pipeline.

---

## 5. KNIME Workflow Layer — Initial Curation

Each workflow performs:

- Deduplication  
- Type casting  
- Timestamp normalization  
- Sorting  
- Merging with historical curated data  
- Schema enforcement  

### Curated Outputs (Parquet)

- GOLD_USD_DAILY_OHLC.snappy.parquet  
- GOLD_USD_HOURLY_OHLC.snappy.parquet  
- DFF.snappy.parquet  
- VIXCLS.snappy.parquet  

These curated files feed downstream ML forecasting pipelines.

---

## 6. End-to-End Summary

This architecture enables:

- Continuous ingestion of gold & macroeconomic data  
- Reliable synchronization and centralized raw storage  
- Event-driven ETL using EventBridge + SQS + KNIME  
- Automated curated Parquet datasets  
- Zero manual intervention for hourly/daily updates  

This serves as the backbone of the Mountaingoat Gold Prediction Platform.
