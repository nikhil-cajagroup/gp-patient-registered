# Patients Registered at a GP Practice – Data Guide (for RAG / LangChain)

This document describes how the **GP registered patient** dataset has been modelled in your analytics environment and how it can be used effectively with **RAG / LangChain** or other LLM tooling.

The source is NHS England’s monthly **“Patients Registered at a GP Practice”** publication, which is extracted as a snapshot from the Personal Demographics Service (PDS). It provides counts of patients registered with GP practices in England by geography, sex and age.

Your Athena database is:

```sql
test-patient-registered
```

It currently contains the following tables:

- `lsoa_by_sex`
- `practice_all`
- `practice_map`
- `practice_quinary_age`
- `practice_single_age_female`
- `practice_single_age_male`
- `single_age_regions`

Each table is described in detail below, followed by join diagrams, example queries and guidance for using this data with LangChain.

---

## 1. Conceptual view of the dataset

### 1.1 What the data represents

- A **monthly snapshot** of patients registered with GP practices in England.
- Counts, not individual-level records (i.e. each row is an aggregated cell: a geography × sex × age combination).
- Data can be analysed:
  - At **practice** level (most granular organisation).
  - Rolled up to **PCN, Sub-ICB, ICB, NHS England commissioning region**.
  - For some tables, down to **LSOA** level (approx. neighbourhood of ~1,500 people).

### 1.2 Common concepts across tables

Across all tables you will repeatedly see:

- **Time**: `extract_date`, `period` (e.g. `2025-01`), `year`
- **Organisation / geography codes**: practice, PCN, CCG / ICB, region
- **Demographics**:
  - `sex` (ALL / MALE / FEMALE)
  - `age` (single year or ALL)
  - `age_group_5` (5‑year quinary bands)
- **Measure**:
  - `n_patients`: the count of registered patients in that cell.

These common fields are what allow you to **join** tables together or compare different levels of aggregation.

---

## 2. Table reference

### 2.1 `lsoa_by_sex`

**Grain (one row =):**  
LSOA × practice × sex × snapshot period.

**Key columns (from your sample):**

- `publication` – source identifier (e.g. `GP_PRAC_PAT_LIST`).
- `extract_date` – date snapshot was taken (e.g. `2022-04-01`).
- `practice_code` / `practice_name` – GP practice ODS code and name.
- `lsoa_code` – 2011 LSOA code where the patient lives.
- `sex` – `MALE`, `FEMALE`, sometimes `ALL`.
- `n_patients` – number of registered patients in this practice–LSOA–sex cell.
- `period` – month string (e.g. `2022-04`).
- `year` – year as integer.

**Usage notes**

- Best table if you need a **practice catchment map by LSOA**, for example “which LSOAs supply patients to this practice?”
- Also allows **local population profiling** by linking LSOA to deprivation indices, rural/urban classification, etc.
- For RAG you might describe each LSOA–practice–sex cell in words, but in practice you will usually summarise metrics in SQL and feed the numeric output to the LLM rather than indexing every row.

---

### 2.2 `practice_all`

**Grain:**  
Practice × sex × age band × period.

**Key columns:**

- `publication`, `extract_date`
- `type` – organisation type, usually `GP` for practices.
- `ccg_code`, `ons_ccg_code` – legacy CCG identifiers (still present in many series).
- `code` – practice code (ODS); effectively the same entity as `practice_code` in other tables.
- `postcode` – practice postcode.
- `sex` – `ALL`, `MALE`, `FEMALE`.
- `age` – either `ALL` or broad band; in your sample it is `ALL`.
- `n_patients` – number of registered patients for that combination.
- `period`, `year`.

**Usage notes**

- This table gives the **headline registered list size per practice** (and sometimes per age band).
- Good starting point for:
  - “What is the current patient list size of practice X?”
  - “How has the registered list size changed over time?”
- For RAG you can pre‑compute trends (e.g. list size growth over the last 12 months) and store short textual summaries per practice.

---

### 2.3 `practice_map`

**Grain:**  
Practice × period (organisational mapping).

**Key columns:**

- `publication`, `extract_date`
- `practice_code`, `practice_name`
- `practice_postcode`
- `pcn_code`, `pcn_name`
- `ons_ccg_code`, `ccg_code`, `ccg_name`
- `ons_stp_code`, `stp_code`, `stp_name` – historic STP / ICB mappings.
- `ons_comm_region_code`, `comm_region_code`, `comm_region_name`
- `period`, `year`.

**Usage notes**

- A **pure mapping table** – it contains **no patient counts**.
- Use it to attach organisational structure to any practice‑level metric:
  - Join from tables that have a practice code (e.g. `practice_all`, `lsoa_by_sex`) to get PCN/ICB/region.
- Helps you answer questions such as:
  - “Which practices are in this PCN or ICB?”
  - “Which ICB does this practice belong to at a given date?”
- Because organisational structures change over time, always join using both **practice code AND period** where possible.

---

### 2.4 `practice_quinary_age`

**Grain:**  
Organisation (practice / PCN / region, etc.) × 5‑year age band × sex × period.

From your sample row:

- `org_type` – type of organisation (e.g. `Comm Region`, `Practice`, `PCN`, `ICB`).
- `org_code` – organisation identifier (e.g. region code `Y56`).
- `ons_code` – matching ONS code for that organisation level.
- `postcode` – usually blank except at practice level.
- `sex` – `MALE`, `FEMALE`, `ALL`.
- `age_group_5` – five‑year band (e.g. `0-4`, `5-9`, `70-74`, or `ALL`).
- `n_patients` – count for that cell.
- `period`, `year`.

**Usage notes**

- Best suited to **age structure analysis**, such as:
  - “What proportion of patients in this region are aged 65+?”
  - “How does the age profile of practice A compare to the regional profile?”
- For LLM/RAG you can pre‑generate short descriptions:
  - “This region has 20% of registered patients aged 65 or over, compared with 18% nationally.”

---

### 2.5 `practice_single_age_female` and `practice_single_age_male`

These two tables are structurally identical; they differ only in the **fixed sex** dimension.

**Grain:**  
Organisation (usually practice) × single year of age × period.

Common columns (from your samples):

- `extract_date`
- `ccg_code`, `ons_ccg_code`
- `org_code` – typically practice ODS code when `type` is GP.
- `postcode` – practice postcode.
- `sex` – here either `FEMALE` or `MALE` for the respective table.
- `age` – text label for age band (often a number as a string, or `ALL`).
- `age_num` – numeric age when available.
- `n_patients` – count for that cell.
- `period`, `year`.

**Usage notes**

- Provide **single year of age** detail, which is helpful for:
  - Detailed age‑specific rates (e.g. vaccination coverage, QOF prevalence denominators when linked to other data sources).
  - Fine‑grained analysis near age thresholds (e.g. 16, 18, 65).
- Because there is one table per sex, you may want to build a **view** that union‑all’s them with an explicit `sex` column to simplify querying.

Example union view:

```sql
CREATE OR REPLACE VIEW practice_single_age_all AS
SELECT
  extract_date,
  ccg_code,
  ons_ccg_code,
  org_code,
  postcode,
  'FEMALE' AS sex,
  age,
  age_num,
  n_patients,
  period,
  year
FROM "test-patient-registered"."practice_single_age_female"
UNION ALL
SELECT
  extract_date,
  ccg_code,
  ons_ccg_code,
  org_code,
  postcode,
  'MALE' AS sex,
  age,
  age_num,
  n_patients,
  period,
  year
FROM "test-patient-registered"."practice_single_age_male";
```

---

### 2.6 `single_age_regions`

**Grain:**  
Region (or other high‑level organisation) × single year of age × period.

Your sample row:

- `publication`
- `extract_date`
- `org_type` – here `Comm Region`.
- `org_code` – region code (e.g. `Y56`).
- `ons_code` – ONS region code (`E40000003`).
- `sex` – `MALE`, `FEMALE`, `ALL`.
- `age` – usually a single year of age or `ALL`.
- `age_num` – numeric age where present.
- `n_patients` – regional count for that age/sex.
- `period`, `year`.

**Usage notes**

- This is effectively the **regional aggregation** of the single‑age practice data.
- Useful for benchmarking practice/PCN numbers against regional totals.
- In RAG, this can underpin natural‑language summaries like “Within North East and Yorkshire, there are 11.1 million registered patients in total, including 2.0 million aged 65+.”

---

## 3. Relationships between tables

All tables are ultimately linked by **time** and **organisation codes**.

High‑level join patterns:

1. **Practice to organisation context**  
   - Join any table with a practice code to `practice_map` to get PCN / ICB / region.

   ```sql
   SELECT a.period,
          a.code AS practice_code,
          a.n_patients,
          m.pcn_code,
          m.icb_code,
          m.comm_region_name
   FROM "test-patient-registered"."practice_all" a
   LEFT JOIN "test-patient-registered"."practice_map" m
     ON a.code = m.practice_code
    AND a.period = m.period;
   ```

2. **Practice headline vs age breakdown**  
   - Compare total list size from `practice_all` with sums by age from `practice_quinary_age` or the single‑age tables.

3. **Practice catchment vs practice list size**  
   - Use `lsoa_by_sex` to see where registered patients live, but check that totals by practice roughly match `practice_all` for consistency (allowing for counting rules).

Because this is official published data, standard caveats apply (e.g. over‑registration, people registered but not resident, delays in de‑registration). When using this data in narratives, it’s worth acknowledging that these are **registration counts**, not a perfect measure of the resident population.

---

## 4. Example analytical questions

Here are some concrete questions you can answer with this schema.

1. **List size trend for a practice**  
   - Table: `practice_all`  
   - Question: “How has the registered list size of practice A84002 changed from 2021 to 2025?”

2. **Age profile of a GP practice**  
   - Tables: `practice_quinary_age` (or single‑age tables) + `practice_map`.  
   - Question: “What proportion of patients at THE DENSHAM SURGERY are over 65, and how does this compare to its region?”

3. **Catchment by LSOA**  
   - Table: `lsoa_by_sex`.  
   - Question: “Which ten LSOAs contribute the most registered patients to practice A81001?”

4. **Regional patient totals**  
   - Table: `single_age_regions`.  
   - Question: “How many patients are registered in the North East and Yorkshire commissioning region?”

5. **Gender balance**  
   - Tables: `practice_all`, `practice_single_age_male`, `practice_single_age_female`.  
   - Question: “Is the practice’s registered population more female than male, particularly in older age groups?”

---

## 5. Using this dataset with LangChain / RAG

### 5.1 General approach

Because this dataset is **large and numeric**, you usually don’t want to index every raw row into a vector store. Instead:

1. **Do the heavy lifting in SQL** (Athena / Trino / Presto) to aggregate and filter.
2. **Return tidy numeric tables** to the LLM.
3. Let the LLM **explain, compare and summarise** the numeric outputs in natural language.

Possible patterns:

- **SQL‑generating agent**: Use LangChain’s SQL Database toolkit pointing at your Athena catalog. The LLM writes SQL from natural‑language questions, executes it and explains the result.
- **Pre‑generated text chunks**: For each practice or region, pre‑compute a short textual summary (list size, age profile, recent growth) and store that in a document store. The LLM retrieves and reuses those summaries.

### 5.2 Chunking strategy for documentation‑style RAG

If you decide to build a documentation corpus describing each practice or region:

- Create one document per **practice** or **region**, including:
  - Basic identifiers from `practice_map` (codes, names, PCN, ICB, region, postcode).
  - Current list size and 12‑month change from `practice_all`.
  - Age/sex profile from `practice_quinary_age` or the single‑age tables.
  - Any notable points (e.g. unusually high proportion of older patients).

- Each document can be chunked into sections (identity, overall list size, age profile, catchment LSOAs).  
- These chunks can be embedded (e.g. with OpenAI text-embedding models) for semantic retrieval.

### 5.3 Example RAG prompt for a LangChain agent

System prompt (sketch):

> You are an analyst familiar with NHS England’s “Patients Registered at a GP Practice” dataset.  
> Use the SQL tables in the `test-patient-registered` database to answer questions about registered patient counts, age profiles and geographies.  
> Always state the period (month/year) you are using in your answer.  
> Explain that figures are counts of registered patients, not necessarily the resident population.

User question example:

> “For practice A84002, what is the current registered list size and how has it changed over the last three years? Also comment on its age profile compared to the national picture.”

The agent would:

1. Generate SQL queries against `practice_all` for list size over time.
2. Query `practice_quinary_age` or single‑age tables for age structure.
3. Optionally pull regional/national comparators from `single_age_regions`.
4. Produce a narrative answer with tables/percentages.

---

## 6. Quality and caveats (for LLM answers)

When building assistants on top of this dataset, it’s important they mention key caveats where relevant:

- These are **registration counts**, not a true census of where people live.
- Some people may be double‑registered or registered at a practice far from where they actually live.
- LSOA allocations rely on address postcodes; a small proportion of patients cannot be assigned to an LSOA.
- Organisational boundaries change over time, so comparisons across long periods should be interpreted carefully.

You can bake these points into your system prompt so that the assistant automatically frames answers appropriately.

---

## 7. Quick reference: tables and primary keys

| Table name                         | Grain (primary key)                                                  |
|-----------------------------------|----------------------------------------------------------------------|
| `lsoa_by_sex`                     | (`practice_code`, `lsoa_code`, `sex`, `period`)                      |
| `practice_all`                    | (`code`, `sex`, `age`, `period`)                                     |
| `practice_map`                    | (`practice_code`, `period`)                                          |
| `practice_quinary_age`           | (`org_type`, `org_code`, `sex`, `age_group_5`, `period`)             |
| `practice_single_age_female`     | (`org_code`, `age_num`, `period`) with implicit `sex = 'FEMALE'`     |
| `practice_single_age_male`       | (`org_code`, `age_num`, `period`) with implicit `sex = 'MALE'`       |
| `single_age_regions`             | (`org_type`, `org_code`, `sex`, `age_num`, `period`)                 |

This guide should give both analysts and LLM agents a clear mental model of how the **patients registered at a GP practice** data is structured in your `test-patient-registered` database and how to work with it safely and effectively.
