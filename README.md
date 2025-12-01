# Patients Registered at a GP Practice – AWS Ingestion

This repository contains code to ingest the **Patients Registered at a GP Practice** dataset into an AWS data lake.

The dataset provides monthly snapshots of the number of patients registered with each GP practice in England.  [oai_citation:35‡NHS England Digital](https://digital.nhs.uk/data-and-information/publications/statistical/patients-registered-at-a-gp-practice?utm_source=chatgpt.com)

---

## Source dataset

- **Name:** Patients Registered at a GP Practice  
- **Owner:** NHS England / NHS Digital  [oai_citation:36‡NHS England Digital](https://digital.nhs.uk/data-and-information/publications/statistical/patients-registered-at-a-gp-practice?utm_source=chatgpt.com)  
- **Description:** Data extracted from the Primary Care Registration database within the Personal Demographics Service (PDS).  [oai_citation:37‡NHS England Digital](https://digital.nhs.uk/data-and-information/publications/statistical/patients-registered-at-a-gp-practice/april-2025?utm_source=chatgpt.com)  
- **Licence:** Usually OGL – confirm on the publication page.

---

## Geographic coverage & granularity

- **Coverage:** England  
- **Granularity:**  
  - GP practice  
  - Aggregated to CCG/ICB, region, and national levels in some tables.

---

## Time coverage & refresh

- **Monthly** snapshots (e.g. 1 April 2025 snapshot).  [oai_citation:38‡NHS England Digital](https://digital.nhs.uk/data-and-information/publications/statistical/patients-registered-at-a-gp-practice/april-2025?utm_source=chatgpt.com)  
- Long time series available back to 2020 and earlier.  [oai_citation:39‡NHS England Digital](https://digital.nhs.uk/data-and-information/publications/statistical/patients-registered-at-a-gp-practice?utm_source=chatgpt.com)  

---

## This project’s data model

- **Bronze layer**  
  - Raw monthly snapshot CSVs uploaded to S3, partitioned by `snapshot_year` and `snapshot_month`.  

- **Silver layer**  
  - `fact_gp_registered_patients` with fields like:
    - `snapshot_date`  
    - `practice_code`, `practice_name`  
    - `icb_code`, `region`  
    - `age_band`, `sex` (if included)  
    - `registered_population`  

---

## Repository contents

- `Untitled.ipynb` – Bronze ingestion notebook (rename to `bronze_gp_patient_registered.ipynb` or similar).  

---

## How to use

1. Download monthly CSVs from the “Patients Registered at a GP Practice” publication series.  [oai_citation:40‡NHS England Digital](https://digital.nhs.uk/data-and-information/publications/statistical/patients-registered-at-a-gp-practice?utm_source=chatgpt.com)  
2. Run the Bronze notebook to ingest snapshots to S3.  
3. Build Silver tables that can be joined to workforce, appointments, and outcome datasets via practice/ICB codes.

---

## Attribution

Please attribute NHS England for the “Patients Registered at a GP Practice” statistics as per the publication guidance.  [oai_citation:41‡NHS England Digital](https://digital.nhs.uk/data-and-information/publications/statistical/patients-registered-at-a-gp-practice?utm_source=chatgpt.com)
