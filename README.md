# Subscription Mini Project - Technical Documentation

## Executive Summary
This document provides comprehensive technical documentation for the **Subscription Mini Project**, a data engineering solution implementing a medallion architecture (Bronze-Silver-Gold) to process and analyze subscription-based business data. The project enables tracking of customer subscriptions, revenue metrics, and key performance indicators (KPIs) for business intelligence.

---

## Table of Contents
1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Data Sources](#data-sources)
4. [Bronze Layer (Raw Data Ingestion)](#bronze-layer)
5. [Silver Layer (Data Cleaning & Transformation)](#silver-layer)
6. [Gold Layer (Business-Ready Analytics)](#gold-layer)
7. [Data Model](#data-model)
8. [KPI Analyses](#kpi-analyses)
9. [Technical Implementation](#technical-implementation)
10. [Project Structure](#project-structure)

---

## 1. Project Overview

### Business Context
The project analyzes subscription-based revenue data to provide insights into:
- Customer acquisition and churn patterns
- Monthly Recurring Revenue (MRR) trends
- Customer lifetime value
- Product adoption and expansion
- Revenue retention metrics

### Technology Stack
- **Platform**: Databricks on AWS
- **Processing Engine**: Apache Spark (PySpark)
- **Storage Format**: Delta Lake
- **Catalog**: Unity Catalog (Bronze, Silver, Gold)
- **Language**: Python/PySpark
- **Architecture Pattern**: Medallion Architecture (Bronze-Silver-Gold)

### Key Objectives
1. Ingest raw subscription data from external sources
2. Clean and standardize data with proper type conversions
3. Create dimensional model for analytics
4. Calculate subscription metrics (MRR, revenue retention)
5. Generate actionable business insights through KPI dashboards

---

## 2. Architecture

### Medallion Architecture Overview

```
External Data Sources (Azure Blob Storage)
           |
           v
    ┌──────────────────┐
    │  BRONZE LAYER    │  <- Raw data ingestion (as-is)
    │  bronze_catalog  │
    └──────────────────┘
           |
           v
    ┌──────────────────┐
    │  SILVER LAYER    │  <- Data cleaning & transformation
    │  silver_catalog  │
    └──────────────────┘
           |
           v
    ┌──────────────────┐
    │   GOLD LAYER     │  <- Business-ready dimensional model
    │   gold_catalog   │
    │                  │
    │  Facts:          │
    │  - Subscription  │
    │                  │
    │  Dimensions:     │
    │  - Customer      │
    │  - Product       │
    └──────────────────┘
           |
           v
    ┌──────────────────┐
    │  KPI ANALYSIS    │  <- Business intelligence & reporting
    └──────────────────┘
```

### Data Flow
1. **Ingestion**: Raw CSV files from Azure Blob Storage → Bronze tables
2. **Cleaning**: Bronze tables → Silver tables (remove metadata, standardize formats)
3. **Enrichment**: Join multiple silver tables, calculate derived metrics
4. **Modeling**: Silver tables → Gold dimensional model (facts & dimensions)
5. **Analysis**: Gold tables → KPI calculations and visualizations

---

## 3. Data Sources

### Source System
- **Location**: Azure Blob Storage (`subscripton_mini_project.azure_blob_storage`)
- **Ingestion Tool**: Fivetran (CDC-based ingestion)
- **Format**: CSV files

### Source Tables

| Table Name | Description | Key Columns |
|------------|-------------|-------------|
| **customer** | Customer master data | customer_id, customer_name, country, industry_type, account_created_date, is_active |
| **opportunity** | Subscription opportunities/contracts | opportunity_id, customer_id, product_id, employee_id, start_date, end_date, contract_term, revenue_amount, close_status |
| **product** | Product catalog | product_id, product_name, plan_name, billing_cycle, list_price, is_active |
| **country** | Country and currency mapping | country_code, country_name, currency_code |
| **employee** | Employee/sales rep data | employee_id, employee_name, role, region, hire_date, is_active_flag |
| **fx_rate** | Foreign exchange rates | currency_code, fx_rate_to_gbp, effective_date |

---

## 4. Bronze Layer (Raw Data Ingestion)

### Purpose
Store raw, unmodified data from source systems with minimal transformation. Preserve data lineage through Fivetran metadata columns.

### Catalog: `bronze_catalog`
### Schema: `default`

### Implementation Details

**Notebook**: `/subscription_mini_project/subscription_de_mini_project/subscription_bronze/raw`

#### Process
1. Create Bronze catalog and schema
2. Ingest data using `CREATE TABLE AS SELECT` (CTAS) from Azure Blob Storage
3. Preserve Fivetran metadata columns:
   - `_file`: Source file path
   - `_line`: Line number in source file
   - `_modified`: Last modification timestamp
   - `_fivetran_synced`: Fivetran sync timestamp

#### Bronze Tables Created

| Table Name | Row Count (Approx) | Key Characteristics |
|------------|-------------------|---------------------|
| `bronze_catalog.default.customer_table` | Customer records | Includes Fivetran metadata |
| `bronze_catalog.default.opportunity_table` | Subscription records | Raw string dates and revenue |
| `bronze_catalog.default.product_table` | Product records | List prices in various currencies |
| `bronze_catalog.default.country_table` | Country mappings | ISO country codes |
| `bronze_catalog.default.employee_table` | Employee records | Sales rep information |
| `bronze_catalog.default.fixRate_table` | FX rates | GBP conversion rates |

#### Code Example
```python
# Create catalog
spark.sql("CREATE CATALOG IF NOT EXISTS bronze_catalog COMMENT 'Catalog for raw, bronze layer data'")

# Create schema
spark.sql("CREATE SCHEMA IF NOT EXISTS bronze_catalog.default")

# Ingest raw data
spark.sql("""
    CREATE TABLE IF NOT EXISTS bronze_catalog.default.opportunity_table 
    AS SELECT * FROM subscripton_mini_project.azure_blob_storage.opportunity
""")
```

---

## 5. Silver Layer (Data Cleaning & Transformation)

### Purpose
Clean, standardize, and enrich data. Remove technical metadata, fix data types, handle nulls, and join reference data.

### Catalog: `silver_catalog`
### Schema: `default`

### Implementation Details

**Folder**: `/subscription_mini_project/subscription_de_mini_project/subscription_silver/`

**Notebooks**:
- `Customer_Table_Cleaning`
- `Opportunity_Table_Cleaning`
- `Product_Table_Cleaning`
- `Employee_Table_Cleaning`
- `CountryTable_Cleaning`
- `FxRate_Table_Cleaning`

### Transformations Applied

#### 1. Metadata Removal
```python
fivetran_ingested_column_to_remove = [
    "_file", 
    "_fivetran_synced", 
    "_modified",
    "_line"
]

def remove_fivetran_ingested_column(df):
    for column in fivetran_ingested_column_to_remove:
        df = df.drop(column)
    return df
```

#### 2. Opportunity Table Transformations

**Key Transformations**:
- **Date Standardization**: Convert string dates to proper DATE type
  ```python
  df = df.withColumn("start_date", to_date(col("start_date"), "dd-MM-yyyy"))
  df = df.withColumn("end_date", to_date(col("end_date"), "dd-MM-yyyy"))
  ```

- **Revenue Cleaning**: Remove currency symbols and convert to numeric
  ```python
  df = df.withColumn(
      "revenue_amount",
      regexp_replace(col("revenue_amount"), '[$£]', '')
  )
  df = df.withColumn(
      "revenue_amount",
      col("revenue_amount").cast("double")
  )
  ```

- **Status Standardization**: Uppercase close_status for consistency
  ```python
  df = df.withColumn("close_status", upper(col("close_status")))
  ```

- **Timestamp Conversion**:
  ```python
  df = df.withColumn(
      "created_timestamp", 
      to_timestamp(col("created_timestamp"), "dd-MM-yyyy HH:mm")
    )
  ```

- **Deduplication**:
  ```python
  df = df.dropDuplicates()
  ```

#### 3. Data Enrichment

**Join customer data with country and FX rate information**:
```python
opportunity_df = opportunity_df.alias("o") \
    .join(customer_df.alias("c"), col("o.customer_id") == col("c.customer_id"), "left") \
    .select("o.*", "c.country").alias("co") \
    .join(country_master_df.alias("cm"), col("co.country") == col("cm.country_code"), "left") \
    .select("co.*", "cm.currency_code").alias("coc") \
    .join(fx_rate_df.alias("fx"), col("coc.currency_code") == col("fx.currency_code"), "left") \
    .select("coc.*", "fx.fx_rate_to_gbp")
```

#### 4. Calculated Fields

**Revenue in GBP**:
```python
opportunity_df = opportunity_df.withColumn(
    "revenue_in_gpb",
    col("revenue_amount") * col("fx_rate_to_gbp")
)
```

**Contract Months**:
```python
opportunity_df = opportunity_df.withColumn(
    "contract_months",
    when(col("contract_term") == "Monthly", 1)
    .when(col("contract_term") == "Yearly", 12)
)
```

**Monthly Recurring Revenue (MRR)**:
```python
opportunity_df = opportunity_df.withColumn(
    "mrr_in_gpb",
    col("revenue_in_gpb") / col("contract_months")
)
```

### Silver Tables Schema

#### `silver_catalog.default.opportunity_table`
| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| opportunity_id | bigint | Unique identifier |
| customer_id | bigint | Foreign key to customer |
| product_id | bigint | Foreign key to product |
| employee_id | bigint | Foreign key to employee |
| start_date | date | Subscription start date |
| end_date | date | Subscription end date |
| contract_term | string | Monthly/Yearly |
| revenue_amount | double | Revenue in local currency |
| close_status | string | WON/ACTIVE/LOST |
| created_timestamp | timestamp | Record creation time |
| country | string | Country code |
| currency_code | string | Currency code |
| fx_rate_to_gbp | double | FX conversion rate |
| revenue_in_gpb | double | **Calculated**: Revenue in GBP |
| contract_months | int | **Calculated**: 1 or 12 months |
| mrr_in_gpb | double | **Calculated**: Monthly recurring revenue |

---

## 6. Gold Layer (Business-Ready Analytics)

### Purpose
Create optimized dimensional model following star schema design for analytics and reporting.

### Catalog: `gold_catalog`
### Schema: `default`

### Implementation Details

**Folders**:
- `/subscription_mini_project/subscription_de_mini_project/subscription_gold/Facts/`
- `/subscription_mini_project/subscription_de_mini_project/subscription_gold/Dimensions/`

### Dimensional Model Design

```
              ┌────────────────┐
              │  dim_customer   │
              └────────┬────────┘
                       │
                       │ customer_id
                       │
              ┌────────▼────────┐
       ┌──────┤  fact_          │◄──────┐
       │      │  subscription   │       │
       │      │  _table         │       │
       │      └────────┬────────┘       │
       │               │                │
       │ employee_id   │ product_id     │
       │               │                │
┌──────▼──────┐       │        ┌───────▼────────┐
│             │       │        │  dim_product   │
│ (           │       │        └────────────────┘
│ dim_employee)│      │
└─────────────┘       │
                      │
              ┌───────▼───────┐
              │  (            │
              │  dim_date)    │
              └───────────────┘
```

### Fact Tables

#### `gold_catalog.default.fact_subscription_table`

**Purpose**: Central fact table capturing all subscription events and metrics.

**Notebook**: `/subscription_gold/Facts/facts_all`

**Grain**: One row per subscription opportunity

**Schema**:
| Column Name | Data Type | Description | Type |
|-------------|-----------|-------------|------|
| opportunity_id | bigint | Primary key | Key |
| customer_id | bigint | Foreign key to dim_customer | FK |
| product_id | bigint | Foreign key to dim_product | FK |
| employee_id | bigint | Foreign key to employee | FK |
| start_date | date | Subscription start | Dimension |
| end_date | date | Subscription end | Dimension |
| contract_months | int | Contract duration | Measure |
| Created_Timestamp | timestamp | Record creation | Dimension |
| Revenue_Local | double | Revenue in local currency | Measure |
| Revenue_GBP | double | Revenue in GBP | **Measure** |
| MRR_GBP | double | Monthly recurring revenue | **Measure** |
| FX_Rate_Used | double | Applied FX rate | Measure |
| close_status | string | WON/ACTIVE/LOST | Dimension |

**Source**:
```python
fact_subscription_df = opportunity_df.select(
    col("close_status"),
    col("opportunity_id"),
    col("customer_id"),
    col("product_id"),
    col("employee_id"),
    col("start_date"),
    col("end_date"),
    col("contract_months"),
    col("created_timestamp").alias("Created_Timestamp"),
    col("revenue_amount").alias("Revenue_Local"),
    col("revenue_in_gpb").alias("Revenue_GBP"),
    col("mrr_in_gpb").alias("MRR_GBP"),
    col("fx_rate_to_gbp").alias("FX_Rate_Used")
)
```

### Dimension Tables

**Notebook**: `/subscription_gold/Dimensions/dim_all`

#### 1. `gold_catalog.default.dim_customer`

**Purpose**: Customer master dimension with geographic and business attributes.

**Schema**:
| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| customer_id | bigint | Primary key |
| customer_name | string | Customer name (from customer_internal_id) |
| industry_type | string | Industry classification |
| country_name | string | Full country name |
| currency_code | string | Local currency |
| account_created_date | string | Account creation date |
| is_active | bigint | Active status flag |

**Source**: Joins `silver_catalog.default.customer_table` with `country_table`

```python
dim_customer = df_customer.alias("c").join(
    df_country.alias("ct"),
    col("c.country") == col("ct.country_code"),
    "left"
).select(
    col("c.customer_id").alias("customer_id"),
    col("c.customer_internal_id").alias("customer_name"),
    col("c.industry_type").alias("industry_type"),
    col("ct.country_name").alias("country_name"),
    col("ct.currency_code").alias("currency_code"),
    col("c.account_created_date").alias("account_created_date"),
    col("c.is_active").alias("is_active")
).dropDuplicates(["customer_id"])
```

#### 2. `gold_catalog.default.dim_product`

**Purpose**: Product catalog dimension with pricing and plan details.

**Schema**:
| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| product_id | bigint | Primary key |
| product_name | string | Product name (from product_internal_id) |
| plan_name | string | Basic/Pro/Enterprise/Ultimate |
| billing_cycle | string | Monthly/Yearly |
| created_date | date | Product creation date |
| list_price | double | List price |
| is_active | bigint | Active status flag |

**Product Plans Available**:
- Basic
- Pro
- Enterprise
- Ultimate

**Billing Cycles**:
- Monthly (1 month)
- Yearly (12 months)

---

## 7. Data Model

### Entity Relationship Diagram

```
Customer (1) ────< (M) Subscription (M) >──── (1) Product
   │                      Fact                      │
   │                        │                       │
   │                        │                       │
   │                   (M) >──── (1) Employee      │
   │                                                │
   └──────< Country >───── FX Rate ────────────────┘
```

### Cardinality
- One customer can have many subscriptions (1:M)
- One product can be in many subscriptions (1:M)
- One employee can manage many subscriptions (1:M)
- Each subscription has one customer, product, and employee (M:1)

### Key Metrics Calculated

1. **Monthly Recurring Revenue (MRR)**
   ```
   MRR = Revenue / Contract_Months
   ```

2. **Revenue in GBP**
   ```
   Revenue_GBP = Revenue_Local × FX_Rate_to_GBP
   ```

3. **Contract Duration**
   ```
   Contract_Months = IF(contract_term = "Monthly", 1, 12)
   ```

---

## 8. KPI Analyses

### KPI Notebooks

**Primary Notebook**: `/subscription_gold/KPI_Subscription_Sol` (10 cells)

### 1. Customer Gain/Loss Analysis

**Business Question**: *How many customers did we gain and lose this period?*

**Metrics**:
- Customers Gained (WON/ACTIVE status)
- Customers Lost (LOST status)
- Net Change

**Method**:
```python
result = df.groupBy(year(col("start_date")).alias("year")).agg(
    countDistinct(when(col("close_status").isin(["WON", "ACTIVE"]), col("customer_id"))).alias("customers_gained"),
    countDistinct(when(col("close_status") == "LOST", col("customer_id"))).alias("customers_lost")
).withColumn(
    "net_change", 
    col("customers_gained") - col("customers_lost")
)
```

### 2. Churn Trend Analysis

**Business Question**: *Are we losing more customers now compared to previous periods?*

**Metrics**:
- Customers Lost by Year
- Churn Rate %
- Period-over-Period Change
- % Change vs Previous Year

**Method**: Uses window functions for period comparison
```python
window_spec = Window.orderBy("year")
loss_comparison = loss_trend.withColumn(
    "previous_year_lost",
    lag(col("customers_lost")).over(window_spec)
).withColumn(
    "change_percent",
    round(((col("customers_lost") - col("previous_year_lost")) / col("previous_year_lost")) * 100, 2)
)
```

### 3. High-Value Customer Analysis

**Business Question**: *Who are our highest-value customers by recurring revenue?*

**Metrics**:
- Total MRR per customer
- Total Revenue per customer
- Number of active subscriptions
- Customer details (name, industry, country)

**Method**: Aggregate MRR by customer, join with customer dimension
```python
highest_value_customers = fact_df.filter(col("close_status").isin(["ACTIVE", "WON"])) \
    .groupBy("customer_id").agg(
        _sum("MRR_GBP").alias("total_mrr_gbp"),
        _sum("Revenue_GBP").alias("total_revenue_gbp"),
        count("opportunity_id").alias("active_subscriptions")
    ) \
    .join(dim_customer, "customer_id", "inner")
```

### 4. Customer Expansion Analysis

**Business Question**: *Which customers are expanding their product usage over time?*

**Metrics**:
- Year-over-year product count growth
- Year-over-year MRR growth
- MRR growth percentage

**Criteria for Expansion**:
- Product growth > 0 OR MRR growth > 10%

**Method**: Window functions for YoY comparison
```python
window_spec = Window.partitionBy("customer_id").orderBy("year")

customer_growth = customer_yearly_metrics.withColumn(
    "prev_year_products",
    lag(col("unique_products")).over(window_spec)
).withColumn(
    "mrr_growth_percent",
    round(((col("total_mrr_gbp") - col("prev_year_mrr")) / col("prev_year_mrr")) * 100, 2)
)
```

### 5. Customer Contraction Analysis

**Business Question**: *Which customers are reducing or stopping product usage?*

**Two Analyses**:

a) **Reducing Usage** (Downgrades)
   - Product count decrease
   - MRR decline > 10%
   
b) **Churned Customers** (Complete Stop)
   - Close status = "LOST"
   - Last active year
   - Total lost revenue

### 6. Revenue Retention Analysis

**Business Question**: *How much recurring revenue are we retaining from existing customers?*

**Key Metrics**:
- **Net Revenue Retention (NRR)**: Includes expansion from existing customers
  ```
  NRR = (Retained MRR + Expansion MRR) / Previous Year MRR × 100
  ```
  
- **Gross Revenue Retention (GRR)**: Excludes expansion, pure retention
  ```
  GRR = Retained MRR (capped at previous) / Previous Year MRR × 100
  ```
  
- **Customer Retention Rate**:
  ```
  CRR = Retained Customers / Total Previous Year Customers × 100
  ```

**Method**: Year-over-year cohort analysis
```python
for current_year in range(min_year + 1, max_year + 1):
    previous_year = current_year - 1
    
    # Compare customers from previous year to current year
    retention_analysis = prev_year_customers.join(
        curr_year_customers,
        prev_year_customers["prev_customer_id"] == curr_year_customers["curr_customer_id"],
        "left"
    )
```

### 7. MRR Growth Breakdown

**Business Question**: *How much recurring revenue growth is coming from upgrades versus losses?*

**MRR Movement Categories**:
1. **New**: Customers who didn't exist last year
2. **Expansion**: Existing customers who increased MRR
3. **Retained Flat**: Existing customers with unchanged MRR
4. **Contraction**: Existing customers who decreased MRR
5. **Churned**: Customers who left (MRR → 0)

**Waterfall Analysis**:
```
Starting MRR (FY 2022)
  + New Customer MRR
  + Expansion MRR
  - Contraction MRR
  - Churned MRR
= Ending MRR (FY 2023)
```

**Method**: Categorize each customer's MRR change
```python
categorized_customers = all_customers.withColumn(
    "category",
    when((col("prev_mrr") == 0) & (col("curr_mrr") > 0), "new")
    .when((col("prev_mrr") > 0) & (col("curr_mrr") == 0), "churned")
    .when((col("prev_mrr") > 0) & (col("curr_mrr") > col("prev_mrr")), "expansion")
    .when((col("prev_mrr") > 0) & (col("curr_mrr") < col("prev_mrr")), "contraction")
    .otherwise("retained_flat")
)
```

### 8. Year-to-Date Revenue Comparison

**Business Question**: *How does revenue year-to-date compare with the same period last year?*

**Metrics**:
- YTD Revenue (current year)
- YTD Revenue (previous year)
- Absolute change
- Percentage change
- Monthly breakdown

**Method**: Compare same day-of-year periods
```python
latest_day_of_year = fact_df.filter(year(col("start_date")) == latest_year) \
    .select(max(dayofyear(col("start_date")))).collect()[0]['max_doy']

# Compare both years up to the same day
current_ytd = fact_df.filter(
    (year(col("start_date")) == latest_year) & 
    (dayofyear(col("start_date")) <= latest_day_of_year)
)
```

---

## 9. Technical Implementation

### Code Standards

1. **Naming Conventions**:
   - Catalogs: `{layer}_catalog` (bronze_catalog, silver_catalog, gold_catalog)
   - Tables: `{entity}_table` (customer_table, opportunity_table)
   - Dimensions: `dim_{entity}` (dim_customer, dim_product)
   - Facts: `fact_{subject}_table` (fact_subscription_table)

2. **PySpark Best Practices**:
   - Use column aliasing for joins
   - Apply type casting explicitly
   - Use window functions for temporal analysis
   - Implement deduplication on dimension tables
   - Round numeric results to 2 decimal places

3. **Data Quality**:
   - Remove duplicates
   - Handle nulls appropriately
   - Standardize string values (uppercase)
   - Validate date formats
   - Clean currency symbols

### Performance Optimizations

1. **Delta Lake Benefits**:
   - ACID transactions
   - Time travel capability
   - Schema evolution with `mergeSchema`
   - Optimized file layout

2. **Query Optimizations**:
   - Filter early in transformations
   - Use broadcast joins for small tables
   - Partition large fact tables by date
   - Cache intermediate results when reused

### Error Handling

```python
# Schema evolution for table updates
df.write \
    .format("delta") \
    .mode("overwrite") \
    .option("mergeSchema", "true") \
    .saveAsTable("gold_catalog.default.fact_subscription_table")
```

---

## 10. Project Structure

```
/subscription_mini_project/
├── subscription_de_mini_project/
│   ├── subscription_bronze/
│   │   └── raw                          # Bronze ingestion notebook
│   │
│   ├── subscription_silver/
│   │   ├── Customer_Table_Cleaning      # Silver transformation notebooks
│   │   ├── Opportunity_Table_Cleaning
│   │   ├── Product_Table_Cleaning
│   │   ├── Employee_Table_Cleaning
│   │   ├── CountryTable_Cleaning
│   │   └── FxRate_Table_Cleaning
│   │
│   └── subscription_gold/
│       ├── Facts/
│       │   └── facts_all                # Fact table creation
│       ├── Dimensions/
│       │   └── dim_all                  # Dimension tables creation
│       ├── KPI_Subscription_Sol         # Primary KPI analysis (10 analyses)
│       └── KPI_subscriptions            # Additional KPI workbook
│
└── TECHNICAL_DOCUMENTATION.md           # This document
```

### Catalog Structure

```
bronze_catalog.default/
├── customer_table
├── opportunity_table
├── product_table
├── country_table
├── employee_table
└── fixRate_table

silver_catalog.default/
├── customer_table
├── opportunity_table
├── product_table
├── country_table
├── employee_table
└── fixrate_table

gold_catalog.default/
├── fact_subscription_table    [FACT]
├── dim_customer              [DIMENSION]
└── dim_product               [DIMENSION]
```

---

## Key Achievements

### 1. Architecture
✅ Implemented complete medallion architecture (Bronze-Silver-Gold)  
✅ Established Unity Catalog governance  
✅ Created dimensional model following star schema  
✅ Implemented incremental data processing capability  

### 2. Data Quality
✅ Removed technical metadata columns  
✅ Standardized date formats  
✅ Cleaned revenue data (removed currency symbols)  
✅ Implemented deduplication logic  
✅ Converted data types appropriately  

### 3. Business Logic
✅ Currency conversion to GBP standard  
✅ MRR calculation (Monthly Recurring Revenue)  
✅ Contract duration standardization  
✅ Customer enrichment with geographic data  

### 4. Analytics & KPIs
✅ 8+ comprehensive KPI analyses  
✅ Customer acquisition and churn tracking  
✅ Revenue retention metrics (NRR, GRR)  
✅ Customer expansion/contraction detection  
✅ YoY performance comparisons  
✅ MRR waterfall analysis  

### 5. Code Quality
✅ Modular notebook structure  
✅ Reusable transformation functions  
✅ Clear naming conventions  
✅ Proper documentation in code  
✅ PySpark best practices  

---

## Business Value Delivered

### Revenue Insights
- Track MRR trends over time
- Identify high-value customers
- Measure revenue retention (NRR/GRR)
- Understand revenue growth drivers

### Customer Intelligence
- Customer acquisition vs. churn trends
- Expansion and contraction patterns
- Customer lifetime value tracking
- Industry and geographic segmentation

### Operational Metrics
- Product adoption rates
- Sales performance tracking
- Contract term analysis
- Currency impact analysis

---

## Future Enhancements

### Recommended Next Steps

1. **Additional Dimensions**:
   - `dim_employee`: Sales rep performance analysis
   - `dim_date`: Date dimension for time intelligence
   - `dim_geography`: Geographic hierarchy

2. **Advanced Metrics**:
   - Customer Lifetime Value (CLV)
   - Cohort analysis
   - Predictive churn modeling
   - Product affinity analysis

3. **Automation**:
   - Schedule notebooks as Databricks jobs
   - Implement incremental refresh logic
   - Add data quality checks and alerts
   - Create automated reporting

4. **Visualization**:
   - Build Lakeview dashboards
   - Create executive summary views
   - Implement drill-down capabilities

5. **Performance**:
   - Partition fact table by date
   - Implement Z-ordering on fact table
   - Add table statistics
   - Optimize join strategies

---

## Conclusion

This subscription mini project successfully implements a complete end-to-end data engineering solution using modern data lakehouse architecture on Databricks. The project demonstrates:

- **Technical Proficiency**: Medallion architecture, PySpark transformations, Delta Lake
- **Data Modeling**: Dimensional modeling (star schema) with fact and dimension tables
- **Business Acumen**: Relevant KPIs for subscription business model
- **Code Quality**: Clean, modular, reusable code with best practices
- **Documentation**: Comprehensive technical documentation

The solution provides a scalable foundation for subscription analytics and can be extended to support additional business requirements and advanced analytics use cases.

---

## Technical Contact

**Project Owner**: anmolraturi444@outlook.com  
**Platform**: Databricks on AWS  
**Environment**: Serverless Interactive Cluster  
**Catalog System**: Unity Catalog  

---

**Document Version**: 1.0  
**Last Updated**: March 27, 2026  
**Status**: Ready for Evaluation
