# SQL Data Warehouse Project

A data warehouse built in SQL Server that combines customer, product, and sales data from two source systems (a CRM and an ERP) into one clean, analytics-ready model.

The warehouse follows the Medallion architecture with three layers: **Bronze** (raw data), **Silver** (cleaned data), and **Gold** (business-ready star schema).

---

## Project Goals

- Load raw CSV data from two source systems into SQL Server.
- Clean and standardize the data to fix quality issues such as extra spaces, inconsistent codes, and invalid dates.
- Combine both sources into a single star schema with dimension and fact tables.
- Provide documentation (data catalog and naming conventions) so business users and analysts can understand the data.

---

## Data Architecture

| Layer  | Schema   | Purpose |
|--------|----------|---------|
| Bronze | `bronze` | Stores the raw data exactly as it comes from the source CSV files. |
| Silver | `silver` | Stores cleaned and standardized data. Trims text, maps codes to readable values, and fixes invalid dates. |
| Gold   | `gold`   | Stores business-ready views: two dimension views (`dim_customers`, `dim_products`) and one fact view (`fact_sales`). |

Data flows in one direction: CSV files, then Bronze, then Silver, then Gold.

---

## Source Data

The `datasets/` folder contains six CSV files.

**CRM system (`datasets/source_crm/`)**
- `cust_info.csv`: customer details (name, marital status, gender, create date).
- `prd_info.csv`: product details (name, cost, product line, start and end dates).
- `sales_details.csv`: sales orders (order, ship, and due dates, sales amount, quantity, price).

**ERP system (`datasets/source_erp/`)**
- `CUST_AZ12.csv`: customer birthdate and gender.
- `LOC_A101.csv`: customer country.
- `PX_CAT_G1V2.csv`: product category, subcategory, and maintenance flag.

---

## Repository Structure

```
SQL-Data-Warehouse-Project/
│
├── datasets/                  # Source CSV files from the CRM and ERP systems
│   ├── source_crm/
│   └── source_erp/
│
├── docs/                      # Architecture diagrams, data catalog, naming conventions
│   ├── data_catalog.md        # Descriptions of every table and column in the Gold layer
│   └── naming_conventions.md  # Rules for naming schemas, tables, columns, and procedures
│
├── scripts/                   # SQL scripts, run in the order listed below
│   ├── init_database.sql      # Creates the DataWarehouse database and the bronze, silver, gold schemas
│   ├── bronze/
│   │   ├── ddl_bronze.sql     # Creates the raw Bronze tables
│   │   └── proc_load_bronze.sql   # Stored procedure: loads CSV files into Bronze
│   ├── silver/
│   │   ├── ddl_silver.sql     # Creates the cleaned Silver tables
│   │   └── proc_load_silver.sql   # Stored procedure: cleans Bronze data into Silver
│   └── gold/
│       └── ddl_gold.sql       # Creates the Gold dimension and fact views
│
├── tests/                     # Data quality checks for the Silver and Gold layers
│   ├── quality_checks_silver.sql
│   └── quality_checks_gold.sql
│
├── LICENSE
├── .gitignore
└── README.md
```

---

## Tools Required

- **SQL Server Express** (or any edition of SQL Server)
- **SQL Server Management Studio (SSMS)** to run the scripts
- **Git and GitHub** (optional, for version control)

---

## How to Run the Project

Run the scripts in this order inside SSMS.

1. **Create the database and schemas**
   Run `scripts/init_database.sql`. This creates the `DataWarehouse` database with `bronze`, `silver`, and `gold` schemas.
   **Warning:** if the `DataWarehouse` database already exists, this script drops it and deletes all data.

2. **Create the Bronze tables**
   Run `scripts/bronze/ddl_bronze.sql`.

3. **Create the Bronze load procedure**
   Run `scripts/bronze/proc_load_bronze.sql`. Then load the data:
   ```sql
   EXEC bronze.load_bronze;
   ```
   The procedure truncates each Bronze table and reloads it from the CSV files with `BULK INSERT`.

4. **Create the Silver tables**
   Run `scripts/silver/ddl_silver.sql`.

5. **Create the Silver load procedure and load the data**
   Run `scripts/silver/proc_load_silver.sql`, then:
   ```sql
   EXEC silver.load_silver;
   ```

6. **Create the Gold views**
   Run `scripts/gold/ddl_gold.sql`.

7. **Run the quality checks**
   Run `tests/quality_checks_silver.sql` and `tests/quality_checks_gold.sql`. Most checks should return no rows. Any rows returned point to a data issue that needs attention.

### Important Setup Note

The Bronze load procedure reads the CSV files from a fixed path:
```
C:\sql\dwh_project\datasets\
```
If you cloned or extracted the project somewhere else, either move the project folder to that location or update the file paths in `proc_load_bronze.sql` before running it.

---

## Data Cleaning Highlights (Silver Layer)

- **Whitespace:** names are trimmed of leading and trailing spaces.
- **Missing keys:** customer rows with no customer ID are removed.
- **Marital status:** `S` becomes `Single`, `M` becomes `Married`, and anything else becomes `n/a`.
- **Gender:** `F` becomes `Female`, `M` becomes `Male`, and anything else becomes `n/a`.
- **Customer IDs:** the `NAS` prefix is removed from ERP customer IDs so they match the CRM keys.
- **Product lines:** codes are mapped to names: `M` Mountain, `R` Road, `S` Other Sales, `T` Touring.
- **Product history:** the end date of each product version is calculated from the start date of the next version. The Gold view keeps only current products.
- **Dates:** order, ship, and due dates stored as numbers like `20101229` are converted to real dates. Invalid values (zero or wrong length) become NULL.
- **Sales figures:** if sales is missing, zero, negative, or does not match quantity times price, it is recalculated as quantity times price. If price is missing or invalid, it is derived as sales divided by quantity.

---

## Gold Layer Model

The Gold layer is a star schema with one fact view and two dimension views.

| View | Type | Description |
|------|------|-------------|
| `gold.dim_customers` | Dimension | One row per customer, with name, country, marital status, gender, and birthdate. CRM gender is used first, and ERP gender is the fallback. |
| `gold.dim_products` | Dimension | One row per current product, with name, cost, category, subcategory, and product line. |
| `gold.fact_sales` | Fact | One row per sales order line, with order, shipping, and due dates, sales amount, quantity, and price. It links to both dimensions through surrogate keys. |

Full column descriptions are in [`docs/data_catalog.md`](docs/data_catalog.md).

---

## Naming Conventions

- Use `snake_case` with lowercase letters and underscores.
- Bronze and Silver table names start with the source system name, for example `crm_cust_info` or `erp_loc_a101`.
- Gold names use business terms with prefixes: `dim_` for dimensions and `fact_` for facts.
- Surrogate keys end in `_key`, for example `customer_key` and `product_key`.
- Stored procedures are named `load_<layer>`, for example `bronze.load_bronze`.

See [`docs/naming_conventions.md`](docs/naming_conventions.md) for the full guide.

---

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.
