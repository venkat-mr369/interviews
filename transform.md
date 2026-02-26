
### ✅ **GCP-Based Transformation Architecture: Mapping **

---

### 🔹 **Phase 1: Initial Setup and Data Preparation (F1)**

| Step | Original Component              | GCP Equivalent                                    | Description                                        |
| ---- | ------------------------------- | ------------------------------------------------- | -------------------------------------------------- |
| 1    | Start                           | **Cloud Composer Trigger / Cloud Function**       | Starts the workflow                                |
| 2    | SCI\_CMQ\_mt\_Batch\_Start\_Cmq | **Cloud Function or Dataflow Job Init Step**      | Initializes job, sets up logs, stores metadata     |
| 3    | VW\_DC\_CONTR\_OL\_INS\_CMQ     | **BigQuery View / Dataflow SQL**                  | Filters and transforms control data via BQ views   |
| 4    | Tf\_Stage\_Tables\_Load\_CMQ    | **Cloud Storage → Dataflow → BigQuery (Staging)** | Loads raw files into staging tables for validation |
| 5    | mT\_POL\_ID\_SEQ\_CMQ           | **BigQuery + SEQUENCE() / UUID() functions**      | Generates policy IDs during load                   |
| 6    | sf\_TGT\_RECORDS\_DELETE\_CMQ   | **BigQuery DML / Scheduled Query**                | Cleans up previous records in target tables        |

---

### 🔹 **Phase 2: Policy Core Data Processing (F2)**

| Step | Original Component                  | GCP Equivalent                          | Description                                   |
| ---- | ----------------------------------- | --------------------------------------- | --------------------------------------------- |
| 7    | MT\_SCTFL001\_POLICY\_AMT\_CMQ      | **BigQuery SQL / Dataflow Pipeline**    | Handles policy amount logic, calculations     |
| 8    | mt\_POLICY\_UPD\_TX\_DEL\_FLAG\_CMQ | **BigQuery SQL UPDATE**                 | Updates deletion flags on policy records      |
| 9    | mt\_SCTFL001\_POLICY\_CMQ           | **BigQuery Main Table Insert**          | Inserts core policy master data               |
| 10   | tf\_POLICY\_UPD\_POST\_LOAD\_CMQ    | **BigQuery Post-Load SQL Logic**        | Business rules and final enrichment           |
| 11   | mT\_SCTFL011\_POLICY\_LINE\_CMQ     | **Dataflow to load policy line tables** | Breaks down policies to line-item granularity |

---

### 🔹 **Phase 3: Risk and Coverage Processing (F3)**

| Step | Component                         | GCP Equivalent                               |
| ---- | --------------------------------- | -------------------------------------------- |
| 12   | mt\_SCTFL311\_RISK\_STATE\_cmq    | **BigQuery SQL / Lookup Tables**             |
| 13   | mt\_SCTFL021\_RISK\_LOCATION\_CMQ | **Dataflow / BQ for geo-location handling**  |
| 14   | mt\_SCTFL051\_RISK\_ITEM\_CMQ     | **BigQuery tables for item-wise processing** |

#### ⚡ Parallel Processing Branch 1:

| Step | Component                               | GCP Equivalent                    |
| ---- | --------------------------------------- | --------------------------------- |
| 15a  | mt\_SCTFL061\_RISK\_ITEM\_BUILDING\_CMQ | **BQ: joins + building metadata** |
| 15b  | MT\_SCTFL371\_BUSINESS\_PARTY\_CMQ      | **BQ: joins with party info**     |

#### ⚡ Parallel Processing Branch 2:

| Step | Component                          | GCP Equivalent                   |
| ---- | ---------------------------------- | -------------------------------- |
| 16a  | TF\_SCTFL051\_POLICY\_FEATURE\_cmq | **BigQuery**                     |
| 16b  | tf\_SCTFL041\_COVERAGE\_CMQ        | **BigQuery**                     |
| 17   | tf\_Modifier\_cmq                  | **BigQuery (ratings/modifiers)** |

---

### 🔹 **Phase 4: Final Processing and Closure (F4)**

#### ⚡ Parallel Processing Branch 3:

| Step | Component                                | GCP Equivalent                                            |
| ---- | ---------------------------------------- | --------------------------------------------------------- |
| 18a  | tf\_SCTFL171\_TAX\_SURCHARGE\_CMQ        | **BQ calculations (tax rules per state)**                 |
| 18b  | mT\_SCTFL301\_POLICY\_FORM\_CMQ          | **Document generation using Cloud Run / Cloud Functions** |
| 18c  | mt\_SCTFL431\_PERIL\_SCHD\_RSK\_LOC\_cmq | **BQ joins for peril schedule processing**                |

| Step | Component                       | GCP Equivalent                         |
| ---- | ------------------------------- | -------------------------------------- |
| 19   | mt\_VW\_DC\_CONTROL\_UPD\_CMQ   | **BigQuery UPDATE for audit trail**    |
| 20   | SCI\_CMQ\_mt\_Batch\_Close\_cmq | **Cloud Function or Dataflow job end** |
| 21   | End                             | **Cloud Composer Task Complete**       |

---

## 🗂️ GCP Services Used Summary

| GCP Service                    | Purpose                                      |
| ------------------------------ | -------------------------------------------- |
| **Cloud Storage**              | Ingest raw data (CSV/JSON/parquet etc.)      |
| **Cloud Dataflow**             | Transform data using Apache Beam pipelines   |
| **BigQuery**                   | Store, transform, and analyze large datasets |
| **Cloud Composer**             | Orchestrate the flow of tasks (Airflow)      |
| **Cloud Functions**            | Lightweight trigger logic for jobs           |
| **Cloud Run**                  | Document processing or webhooks (if needed)  |
| **Cloud Logging / Monitoring** | Audit and track job status                   |

---

## 📌 Notes

* **Partitioning** and **clustering** should be applied in BigQuery for performance.
* **Data Validation** can be implemented using **Dataform** or **BQ assertions**.
* **Retry/Failure alerts** via Composer and Cloud Monitoring.
* **IAM roles** control access to staging/final datasets.

---


