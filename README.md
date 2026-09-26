# Clean Banking Marketing Data

Personal loans are a major revenue line for retail banks. Typical two-year personal loan rates in the United Kingdom sit [around 10%](https://www.experian.com/blogs/ask-experian/whats-a-good-interest-rate-for-a-personal-loan/). In September 2022 alone, UK consumers borrowed [roughly £1.5 billion](https://www.ukfinance.org.uk/system/files/2022-12/Household%20Finance%20Review%202022%20Q3-%20Final.pdf)—on that scale, interest income over a two-year term is on the order of **£300 million** for the lending sector.

This project supports a bank that ran a **marketing campaign to promote personal loans**. The bank needs campaign data cleaned, typed, and split into a **stable relational shape** so it can be loaded into **PostgreSQL** and reused for **future campaigns** without rework.

---



## Business requirements



### Background

The bank collected client, campaign, and macroeconomic context in a single export during the campaign. Raw values mix categorical strings (`yes` / `no`, job labels with punctuation), calendar parts (`month`, `day` without year), and numeric indicators. That layout is fine for analysis in a notebook but **not** suitable as a long-lived warehouse schema.

### Goals

1. **Clean** the supplied file `bank_marketing.csv` according to the rules below.
2. **Normalize** one wide table into **three entity-focused CSVs** aligned with how the bank models customers, campaigns, and economics.
3. **Enforce data types and formats** so PostgreSQL can define strict columns and accept **append-only imports** from later campaigns.
4. **Preserve row grain**: one row per `client_id` in each output file (same cardinality as the source, 41,188 clients).



### Out of scope

- *Building* or deploying the PostgreSQL database (downstream consumers use these CSVs as the contract).
- Feature engineering or model training (this is a **data cleaning / conforming** deliverable).
- Changing business definitions (e.g. what counts as campaign success) beyond the mapping rules in this document.



### Success criteria


| Criterion      | Definition of done                                                     |
| -------------- | ---------------------------------------------------------------------- |
| File split     | Exactly three outputs: `client.csv`, `campaign.csv`, `economics.csv`   |
| Schema match   | Column names, order, and types match the specification tables below    |
| Cleaning rules | Every transformation in the “Cleaning requirements” column is applied  |
| Join integrity | `client_id` is unique in each file and aligns across all three         |
| Date format    | `last_contact_date` is `YYYY-MM-DD` with campaign year **2022**        |
| Repeatability  | Pipeline can be re-run on the same source to produce identical outputs |


---



## Source data

**File:** `data/bank_marketing.csv`

**Grain:** one row per client (`client_id`).

**Raw columns:**


| Column                                                                                                                      | Role                                   |
| --------------------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| `client_id`, `age`, `job`, `marital`, `education`, `credit_default`, `mortgage`                                             | Client attributes                      |
| `month`, `day`, `contact_duration`, `number_contacts`, `previous_campaign_contacts`, `previous_outcome`, `campaign_outcome` | Campaign contact and outcomes          |
| `cons_price_idx`, `euribor_three_months`                                                                                    | Economic indicators at time of contact |


---



## Deliverables

After cleaning and splitting, save three CSV files (location: `data/processed`).

### `client.csv`

Customer master for the campaign cohort.


| Column           | Data type | Description                               | Cleaning requirements                                                        |
| ---------------- | --------- | ----------------------------------------- | ---------------------------------------------------------------------------- |
| `client_id`      | `integer` | Client ID                                 | N/A                                                                          |
| `age`            | `integer` | Client's age in years                     | N/A                                                                          |
| `job`            | `object`  | Client's type of job                      | Replace `"."` with `"_"` (e.g. `basic.4y` → not in job; `admin.` → `admin_`) |
| `marital`        | `object`  | Client's marital status                   | N/A                                                                          |
| `education`      | `object`  | Client's level of education               | Replace `"."` with `"_"`; set `"unknown"` to missing (`NaN` / null in CSV)   |
| `credit_default` | `bool`    | Whether the client's credit is in default | Map to boolean: **1** if `"yes"`, **0** otherwise                            |
| `mortgage`       | `bool`    | Whether the client has a housing loan     | Map to boolean: **1** if `"yes"`, **0** otherwise                            |


**PostgreSQL note:** Load as `INTEGER` primary/foreign key on `client_id`; use `BOOLEAN` (or `SMALLINT` CHECK IN (0,1)) for flags; `TEXT` for `job`, `marital`, `education`; nullable `education` where unknown.

---



### `campaign.csv`

Contact history and outcomes for the current and previous campaign.


| Column                       | Data type  | Description                                   | Cleaning requirements                                                                         |
| ---------------------------- | ---------- | --------------------------------------------- | --------------------------------------------------------------------------------------------- |
| `client_id`                  | `integer`  | Client ID                                     | N/A                                                                                           |
| `number_contacts`            | `integer`  | Contact attempts in the **current** campaign  | N/A                                                                                           |
| `contact_duration`           | `integer`  | Last contact duration (seconds)               | N/A                                                                                           |
| `previous_campaign_contacts` | `integer`  | Contact attempts in the **previous** campaign | N/A                                                                                           |
| `previous_outcome`           | `bool`     | Previous campaign outcome                     | **1** if `"success"`, **0** otherwise (includes `"nonexistent"` and other non-success values) |
| `campaign_outcome`           | `bool`     | Current campaign outcome (loan subscribed)    | **1** if `"yes"`, **0** otherwise                                                             |
| `last_contact_date`          | `datetime` | Date of last contact in this campaign         | Build from `day`, `month`, and `year = 2022`; export as `YYYY-MM-DD`                          |


**Date construction:** Source `month` values are lowercase abbreviations (e.g. `may`, `jul`). Parse with a fixed year of **2022** and combine with `day` to form a valid calendar date.

**PostgreSQL note:** `last_contact_date` → `DATE`; boolean flags as above; `client_id` → `INTEGER` referencing `client(client_id)`.

---



### `economics.csv`

Macro indicators linked to each client at campaign time.


| Column                 | Data type    | Description                                | Cleaning requirements |
| ---------------------- | ------------ | ------------------------------------------ | --------------------- |
| `client_id`            | `integer`    | Client ID                                  | N/A                   |
| `cons_price_idx`       | `floatfloat` | Consumer price index (monthly indicator)   | N/A                   |
| `euribor_three_months` | `float`      | Euribor three-month rate (daily indicator) | N/A                   |


**PostgreSQL note:** `DOUBLE PRECISION` or `NUMERIC`; `client_id` → `INTEGER` referencing `client(client_id)`.

---



## Logical data model

```text
client (1) ──< campaign (1)     one campaign row per client for this extract
client (1) ──< economics (1)    one economics row per client for this extract
```

All three tables share `client_id` as the natural key for this campaign file. Future campaigns may append new rows or new campaign tables; keeping types and column names stable avoids migration churn.

---



## Project layout

```text
clean_banking_data/
├── README.md
├── data/
│   ├── bank_marketing.csv          # source (provided)
│   └── processed/                  # optional: cleaned outputs
│       ├── client.csv
│       ├── campaign.csv
│       └── economics.csv
└── etl/
    └── main.ipynb                  # extract → clean → split → export
```

---



## Environment and execution

1. Create and activate a virtual environment in the project root (e.g. `.venv`).
2. Install dependencies:
  ```bash
   pip install pandas numpy matplotlib seaborn openpyxl ipykernel
  ```
3. Open `etl/main.ipynb`, select the `.venv` kernel, and run the pipeline.
4. When reading paths from the notebook, resolve files relative to the **project root** (the notebook often runs with working directory `etl/`).

---



## Data quality checks (recommended)

Before handoff to the database team:

- **Row counts:** `len(client) == len(campaign) == len(economics) == 41_188`
- **Uniqueness:** no duplicate `client_id` in any output
- **Referential alignment:** set of `client_id` identical across all three files
- **Nulls:** `education` nulls only where source was `"unknown"` (after dot replacement, if applicable)
- **Booleans:** only `0`/`1` (or `True`/`False` if exporting with pandas—align with DBA preference; spec uses 0/1 semantics)
- **Dates:** no invalid dates; all `last_contact_date` in 2022 where source month/day imply that year
- **Spot checks:** manual compare of a few `client_id` rows against raw CSV

---



## References

- [Experian — personal loan interest rates (UK context)](https://www.experian.com/blogs/ask-experian/whats-a-good-interest-rate-for-a-personal-loan/)
- [UK Finance — Household Finance Review 2022 Q3](https://www.ukfinance.org.uk/system/files/2022-12/Household%20Finance%20Review%202022%20Q3-%20Final.pdf)

---

