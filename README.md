# Data Cleaning (SQL Server Management Studio)

## Purpose
SSMS was used for:
Data exploration (profiling),
Data type correction,
Cross-checking (reconciliation).

## Combining sheets and correcting the data types

Both sheets were stacked with UNION ALL. Every column is typed explicitly, and SourceSheet keeps track of where each row came from.

sql
-- file.sql
SELECT
    Invoice, StockCode, Description, Quantity,
    COALESCE(
        TRY_CONVERT(datetime2, InvoiceDate, 120),   -- yyyy-mm-dd hh:mi:ss
        TRY_CONVERT(datetime2, InvoiceDate, 105)    -- dd-mm-yyyy
    ) AS InvoiceDate,
    TRY_CONVERT(decimal(10,2), Price) AS Price,
    Customer_ID, Country,
    '2009-2010' AS SourceSheet
INTO dbo.Main_Data
FROM dbo.[Year 2009-2010]

UNION ALL

SELECT
    Invoice, StockCode, Description, Quantity,
    COALESCE(
        TRY_CONVERT(datetime2, InvoiceDate, 120),
        TRY_CONVERT(datetime2, InvoiceDate, 105)
    ) AS InvoiceDate,
    TRY_CONVERT(decimal(10,2), Price) AS Price,
    Customer_ID, Country,
    '2010-2011' AS SourceSheet
FROM dbo.[Year 2010-2011];

<img width="1055" height="767" alt="Data type correction   cross check" src="https://github.com/user-attachments/assets/1286a670-74c1-427f-93b0-a859f315d05e" />

## Why this approach

Choice	Reason
TRY_CONVERT instead of CONVERT	Returns NULL on failure instead of stopping the whole query, so bad values can be counted
COALESCE over two date styles (120, 105)	The two sheets store dates in different text formats; the first style that parses wins
datetime2	Higher precision and a wider range than datetime
DECIMAL(10,2) for Price	Exact currency arithmetic, no floating-point drift
UNION ALL	Keeps every row; duplicates are handled deliberately later, not silently dropped
SourceSheet column	Makes every row traceable to its origin, which is needed to investigate duplicates
4.3 Data exploration (profiling)

Conversion health check. The goal is zero failures.

sql
SELECT COUNT(*) AS Total_Rows     FROM dbo.Main_Data;
SELECT COUNT(*) AS Failed_Dates   FROM dbo.Main_Data WHERE InvoiceDate IS NULL;
SELECT COUNT(*) AS Failed_Prices  FROM dbo.Main_Data WHERE Price IS NULL;

sql
-- Missing Customer ID
SELECT COUNT(*) AS Missing_Customer_ID
FROM dbo.Main_Data WHERE Customer_ID IS NULL;          -- 243,007

-- Negative or zero quantity
SELECT COUNT(*) AS Negative_Qty
FROM dbo.Main_Data WHERE Quantity <= 0;                -- 22,950

-- Missing description
SELECT COUNT(*) AS Missing_Description
FROM dbo.Main_Data WHERE Description IS NULL;          -- 4,382

-- Exact duplicate rows
SELECT Invoice, StockCode, Description, Quantity, InvoiceDate,
       Price, Customer_ID, Country,
       COUNT(*) AS Exact_Duplicate_Count
FROM dbo.Main_Data
GROUP BY Invoice, StockCode, Description, Quantity, InvoiceDate,
         Price, Customer_ID, Country
HAVING COUNT(*) > 1;

<img width="1237" height="767" alt="Data exploration" src="https://github.com/user-attachments/assets/901d3c96-57a4-4dd1-8af2-31bfb8f4ecc5" />

4.5 Export for Python

The typed table is exported from SSMS (Save Results As) to Main_Data.csv. Because the export has no header row, Python reads it with header=None and assigns the 9 column names itself.
