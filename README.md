# azure-sql-sales-pipeline — Project scaffold

This document contains a complete, ready-to-copy project scaffold for a **Sales Data ETL & Reporting Pipeline** that integrates **GitHub**, **Azure SQL Database**, and **Jira**. It includes folder structure, SQL scripts, a GitHub Actions workflow, sample data, `README.md`, and recommended Jira epics/stories/tasks.

---

## Project structure

```
azure-sql-sales-pipeline/
├── data/
│   └── sales_data_sample.csv
├── sql/
│   ├── 01_create_tables.sql
│   ├── 02_create_stored_procs.sql
│   ├── 03_create_views.sql
│   └── 04_cleanup_script.sql
├── .github/
│   └── workflows/
│       └── deploy_sql.yml
├── scripts/
│   └── local_load_sample.ps1
├── README.md
└── jira/
    └── jira-issues-template.md
```

---

## `data/sales_data_sample.csv` (sample)

```
OrderID,OrderDate,Product,Category,Quantity,Price,City,State,PostalCode
1001,2024-01-03,Widget A,Widgets,2,19.99,New York,NY,10001
1002,2024-01-05,Gadget B,Gadgets,1,49.50,Boston,MA,02108
1003,2024-02-10,Widget A,Widgets,5,19.99,Los Angeles,CA,90001
1004,2024-03-12,Thingamajig,Accessories,3,9.99,Austin,TX,73301
1005,2024-04-01,Gadget C,Gadgets,10,29.99,Chicago,IL,60601
```

---

## `sql/01_create_tables.sql`

```sql
-- create staging and final tables
CREATE SCHEMA IF NOT EXISTS sales;

-- staging table for raw imports
CREATE TABLE IF NOT EXISTS sales.Sales_Staging (
    RowID INT IDENTITY(1,1) PRIMARY KEY,
    OrderID INT NULL,
    OrderDate DATE NULL,
    Product NVARCHAR(200) NULL,
    Category NVARCHAR(100) NULL,
    Quantity INT NULL,
    Price DECIMAL(18,2) NULL,
    City NVARCHAR(100) NULL,
    State NVARCHAR(50) NULL,
    PostalCode NVARCHAR(20) NULL,
    ImportedAt DATETIME2 DEFAULT SYSUTCDATETIME()
);

-- final cleaned table
CREATE TABLE IF NOT EXISTS sales.Sales_Final (
    OrderID INT PRIMARY KEY,
    OrderDate DATE NOT NULL,
    Product NVARCHAR(200) NOT NULL,
    Category NVARCHAR(100) NULL,
    Quantity INT NOT NULL,
    Price DECIMAL(18,2) NOT NULL,
    TotalAmount AS (Quantity * Price) PERSISTED,
    City NVARCHAR(100) NULL,
    State NVARCHAR(50) NULL,
    PostalCode NVARCHAR(20) NULL,
    LoadedAt DATETIME2 DEFAULT SYSUTCDATETIME()
);

-- logging / error table
CREATE TABLE IF NOT EXISTS sales.ETL_Log (
    LogID INT IDENTITY(1,1) PRIMARY KEY,
    LogLevel NVARCHAR(10),
    Message NVARCHAR(2000),
    CreatedAt DATETIME2 DEFAULT SYSUTCDATETIME()
);
```

---

## `sql/02_create_stored_procs.sql`

```sql
-- Stored Proc: load from staging into final with simple cleaning and dedupe
CREATE OR ALTER PROCEDURE sales.sp_LoadSalesData
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        -- Insert or update final table from staging
        MERGE INTO sales.Sales_Final AS target
        USING (
            SELECT DISTINCT
                OrderID,
                OrderDate,
                Product,
                Category,
                ISNULL(Quantity,0) AS Quantity,
                ISNULL(Price,0) AS Price,
                City,
                State,
                PostalCode
            FROM sales.Sales_Staging
            WHERE OrderID IS NOT NULL
        ) AS src
        ON target.OrderID = src.OrderID
        WHEN MATCHED THEN
            UPDATE SET
                OrderDate = src.OrderDate,
                Product = src.Product,
                Category = src.Category,
                Quantity = src.Quantity,
                Price = src.Price,
                City = src.City,
                State = src.State,
                PostalCode = src.PostalCode,
                LoadedAt = SYSUTCDATETIME()
        WHEN NOT MATCHED THEN
            INSERT (OrderID, OrderDate, Product, Category, Quantity, Price, City, State, PostalCode)
            VALUES (src.OrderID, src.OrderDate, src.Product, src.Category, src.Quantity, src.Price, src.City, src.State, src.PostalCode);

        -- Optionally truncate staging after load (comment out if you want to keep)
        -- TRUNCATE TABLE sales.Sales_Staging;

        INSERT INTO sales.ETL_Log (LogLevel, Message) VALUES ('INFO', 'sp_LoadSalesData completed successfully');
    END TRY
    BEGIN CATCH
        DECLARE @err NVARCHAR(2000) = ERROR_MESSAGE();
        INSERT INTO sales.ETL_Log (LogLevel, Message) VALUES ('ERROR', @err);
        THROW;
    END CATCH
END
GO
```

---

## `sql/03_create_views.sql`

```sql
-- summary view for analytics
CREATE OR ALTER VIEW sales.vw_Sales_Summary
AS
SELECT
    CONVERT(date, OrderDate) AS OrderDate,
    Category,
    SUM(Quantity) AS TotalQuantity,
    SUM(TotalAmount) AS TotalSales,
    COUNT_BIG(*) AS Orders
FROM sales.Sales_Final
GROUP BY CONVERT(date, OrderDate), Category;
GO

-- top cities
CREATE OR ALTER VIEW sales.vw_Top_Cities
AS
SELECT TOP 50 City, State, SUM(TotalAmount) AS CitySales, SUM(Quantity) AS UnitsSold
FROM sales.Sales_Final
GROUP BY City, State
ORDER BY CitySales DESC;
GO
```

---

## `sql/04_cleanup_script.sql`

```sql
-- clean staging rows older than 30 days
DELETE FROM sales.Sales_Staging WHERE ImportedAt < DATEADD(day, -30, SYSUTCDATETIME());

INSERT INTO sales.ETL_Log (LogLevel, Message) VALUES ('INFO', 'Cleanup run - old staging rows removed');
```

---

## `.github/workflows/deploy_sql.yml`

```yaml
name: Deploy SQL to Azure

on:
  push:
    paths:
      - 'sql/**'

jobs:
  deploy-sql:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python (for sqlcmd)
        uses: actions/setup-python@v4
        with:
          python-version: '3.x'

      - name: Install mssql-tools
        run: |
          sudo apt-get update
          sudo ACCEPT_EULA=Y apt-get install -y curl apt-transport-https gnupg
          curl https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -
          curl https://packages.microsoft.com/config/ubuntu/22.04/prod.list | sudo tee /etc/apt/sources.list.d/mssql-release.list
          sudo apt-get update
          sudo ACCEPT_EULA=Y apt-get install -y mssql-tools unixodbc-dev
          echo 'export PATH="$PATH:/opt/mssql-tools/bin"' >> $GITHUB_ENV

      - name: Execute SQL scripts
        env:
          AZURE_SQL_SERVER: ${{ secrets.AZURE_SQL_SERVER }}     # e.g. myserver.database.windows.net
          AZURE_SQL_DB: ${{ secrets.AZURE_SQL_DB }}             # database name
          AZURE_SQL_USER: ${{ secrets.AZURE_SQL_USER }}
          AZURE_SQL_PASS: ${{ secrets.AZURE_SQL_PASS }}
        run: |
          for f in sql/*.sql; do
            echo "Running $f"
            /opt/mssql-tools/bin/sqlcmd -S $AZURE_SQL_SERVER -d $AZURE_SQL_DB -U $AZURE_SQL_USER -P $AZURE_SQL_PASS -i "$f"
          done
```

**Notes:**

* Add the following GitHub repository secrets: `AZURE_SQL_SERVER`, `AZURE_SQL_DB`, `AZURE_SQL_USER`, `AZURE_SQL_PASS`.
* Alternatively use Azure Service Principal and `azure/login` action for a more secure flow with `sqlpackage` or ARM templates.

---

## `scripts/local_load_sample.ps1`

```powershell
# local helper: bulk load sample CSV into Azure SQL staging using SqlBulkCopy (PowerShell)
# Requires: Install-Module -Name SqlServer

param(
    [string]$CsvPath = "data/sales_data_sample.csv",
    [string]$Conn = "Server=tcp:YOURSERVER.database.windows.net,1433;Database=YOURDB;User ID=YOURUSER;Password=YOURPASSWORD;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
)

Import-Module SqlServer

$dt = Import-Csv -Path $CsvPath

# convert types as needed or adjust headers to match

Write-Host "Loading $($dt.Count) rows into sales.Sales_Staging"

$dt | Write-SqlTableData -ServerInstance $($Conn.Split(';')[0].Replace('Server=tcp:','').Replace(',1433','')) -DatabaseName (($Conn -split ';') | Where-Object { $_ -like 'Database=*' } | ForEach-Object { $_ -replace 'Database=','' }) -SchemaName sales -TableName Sales_Staging -Force

Write-Host "Done."
```

---

## `README.md` (high-level)

```md
# azure-sql-sales-pipeline

Sales Data ETL & Reporting Pipeline — sample project demonstrating integration between GitHub, Azure SQL Database, and Jira.

## What's included
- SQL scripts to create tables, stored procedures, and views
- GitHub Actions workflow to deploy SQL scripts to an Azure SQL Database
- Sample CSV data and a PowerShell helper to push it into staging
- Jira issues template for tracking work

## Setup
1. Create an Azure SQL Database and server.
2. Add required secrets to GitHub repo: AZURE_SQL_SERVER, AZURE_SQL_DB, AZURE_SQL_USER, AZURE_SQL_PASS.
3. Push this repo to GitHub.
4. Run the workflow by pushing changes to `sql/` folder (or trigger manually).
5. Upload sample CSV to staging using `scripts/local_load_sample.ps1`.
6. Execute `EXEC sales.sp_LoadSalesData;` in the database to move data from staging to final.

## Optional
- Configure Azure AD Service Principal + GitHub OIDC for secure deployments
- Add Power BI report pointing to `sales.vw_Sales_Summary`
```

---

## `jira/jira-issues-template.md`

```md
# Jira template: Sales Data ETL & Analytics Pipeline

## Epic: Sales Data ETL & Analytics Pipeline

### Story: Create Azure SQL tables
- Task: Write table creation script (`sql/01_create_tables.sql`)
- Task: Run on Azure Dev DB and validate schema
- Task: Add tests to ensure columns are present

### Story: Build ETL Stored Procedure
- Task: Write `sales.sp_LoadSalesData` (`sql/02_create_stored_procs.sql`)
- Task: Test with sample data
- Task: Add logging and error handling

### Story: Create analytical views
- Task: Create `vw_Sales_Summary` and `vw_Top_Cities` (`sql/03_create_views.sql`)
- Task: Validate against sample dataset

### Story: CI/CD - Deploy SQL from GitHub
- Task: Create GitHub workflow `deploy_sql.yml`
- Task: Add secrets to repo
- Task: Test the workflow

### Story: Data Load Automation
- Task: Build script to bulk upload CSV
- Task: Schedule via GitHub Actions or Azure Function

### Story: Monitoring and Alerts
- Task: Add ETL logging table
- Task: Configure alerting when ETL Log contains ERROR rows
```

---

## How to use this scaffold

1. Copy files into a new GitHub repo (suggested name: `azure-sql-sales-pipeline`).
2. Create the Azure SQL Database and enable firewall to allow GitHub Actions runner IPs or use private endpoints / managed identity.
3. Add repository secrets in GitHub (see `README.md`).
4. Push; the workflow will run and create schema & objects in Azure SQL.
5. Load sample CSV into `sales.Sales_Staging` and run `EXEC sales.sp_LoadSalesData;`.
6. Inspect `sales.Sales_Final` and views.

---

## Next steps I can help with (choose one)

* Create the GitHub repository contents and give you a zipped download
* Generate Power BI sample report steps connecting to views
* Convert the deploy workflow to use Azure Service Principal + `azure/login`
* Add unit tests for SQL (tSQLt examples)
* Add Azure Function to pick CSV from blob storage and push to staging

---


*End of scaffold.*
Deployment test – CI/CD pipeline run
