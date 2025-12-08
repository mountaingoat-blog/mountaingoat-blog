---
layout: post
title: "Gold Prediction – Continuous Ingestion & Curation Platform Architecture"
date: 2025-12-08 12:00:00 +0530
categories: data-platform gold
tags: [gold, ingestion, knime, aws, eventbridge, s3]
---

This architecture powers the continuous ingestion, syncing, and automated curation of all datasets required for Gold Prediction at Mountaingoat.  
It integrates Python-based data fetchers, Lightsail Object Storage, a central AWS S3 lake, EventBridge-based event routing, and KNIME batch workflows for automated dataset curation.

---

## **1. Data Fetching Layer (Lightsail VM)**

A Lightsail VM runs cron-scheduled Python fetchers that pull gold and macroeconomic data from external APIs and store them in Lightsail Object Storage.

### **Gold Fetchers**
- **`fetch_one_day_ohlc_parquet.py`**  
  Fetches XAU/USD *daily* OHLC from TwelveData and writes a **single-row Parquet** file.

- **`fetch_one_hour_ohlc_parquet.py`**  
  Fetches XAU/USD *hourly* OHLC and writes a **single-row Parquet** file.

### **FRED Fetcher**
- **`pull_fred_series_to_s3_csv_dedupe.py`**  
  Downloads macroeconomic series (DFF, DFII10, DGS10, DGS2, DTWEXBGS, VIXCLS, DCOILWTICO).  
  Uploads CSV **only if the latest observation changed**, reducing redundant writes.

2. Sync Layer — Lightsail → AWS S3

A sync script periodically copies new raw files from Lightsail into the central AWS S3 bucket gold-prediction-01.

Sync Script
lightsail_sync_full.sh

Sync Flow

Lightsail bucket → local temporary directory

Local temporary directory → AWS S3 (gold-prediction-01)

AWS S3 becomes the single source of truth for all raw data.

3. Event Routing via EventBridge

EventBridge rules watch S3 prefixes and forward matching object creation events to an SQS queue.

Examples of monitored prefixes:

GOLD_OHLC_Daily_Rate_USD/

GOLD_OHLC_Live_Rate_USD/

us-macro-economic-data/DFF/

us-macro-economic-data/VIXCLS/

Destination queue:

knime-gold-events


Every new file → EventBridge → SQS.

4. KNIME Poller on the KNIME Server

A custom poller script continuously listens to SQS and automatically triggers the corresponding KNIME workflow.

Poller Script:
/mnt/knime-data/poller/bin/knime-gold-poller-single.sh

Poller Responsibilities

Long-poll SQS (20 seconds)

Parse detail.object.key

Map prefix → KNIME workflow directory

Create per-run KNIME temp folder

Execute KNIME in batch mode:

knime -nosplash -nosave \
  -application org.knime.product.KNIME_BATCH_APPLICATION \
  -workflowDir="<workflow>" \
  -reset -consoleLog -Xmx6G

Message Handling Logic

Success: delete message from SQS

Failure: leave message → retry → DLQ

Logs recorded per message for debugging

This makes KNIME fully event-driven.

5. KNIME Workflow Layer — Initial Curation

Each KNIME workflow performs essential curation:

Deduplication

Type conversions

Timestamp parsing and sorting

Schema standardization

Merging with historical curated data

The curated output is saved back to S3 as Parquet.

Final Output Examples

GOLD_USD_DAILY_OHLC.snappy.parquet

GOLD_USD_HOURLY_OHLC.snappy.parquet

DFF.snappy.parquet

VIXCLS.snappy.parquet

These curated datasets are used by ML pipelines for forecasting.

6. Workflow Deployment (Desktop → KNIME Server)
Create ZIP
zip -r US_MACRO_ECONOMIC_VIXCLS_DATA_ETL.zip US_MACRO_ECONOMIC_VIXCLS_DATA_ETL

Upload to server
scp -i gold-etl-ml-01.pem workflow.zip ubuntu@<knime-server>:/mnt/knime-data/

Extract workflow
unzip workflow.zip -d workflows


Once unzipped, the poller automatically triggers the workflow when relevant data arrives.

7. End-to-End Summary

This architecture delivers:

Fully automated ingestion of gold & macroeconomic datasets

Reliable synchronization and centralized raw storage in AWS S3

Event-driven ETL via EventBridge → SQS → KNIME

Continuous curated output in Parquet format

Zero manual intervention
