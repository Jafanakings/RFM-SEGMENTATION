# README: RFM Analysis Using MySQL

## **1. Overview**

This project performs an **RFM (Recency, Frequency, Monetary) analysis** on customer sales data (`SAMPLE_SALES_DATA`) to segment customers based on purchasing behavior.  

**Database used:** `RFM_SALES`  

**Primary goals:**
1. Calculate **Recency, Frequency, and Monetary (RFM) values** for each customer.
2. Compute **RFM scores** using quintiles.
3. Segment customers into categories such as Champions, Loyal Customers, Potential Loyalists, etc.
4. Aggregate monetary values and sales quantities by segment.

---

## **2. Initial Data Exploration**

- View first 20 rows:  
```sql
SELECT * FROM SAMPLE_SALES_DATA LIMIT 20;
```

- Check date range of orders:  
```sql
SELECT MIN(STR_TO_DATE(ORDERDATE, '%d/%m/%y')) AS FIRST_BUSINESS_DAY FROM SAMPLE_SALES_DATA;
SELECT MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y')) AS LAST_BUSINESS_DAY FROM SAMPLE_SALES_DATA;
```

- Current date:  
```sql
SELECT CURDATE();
```

---

## **3. Customer Summary Table**

Calculate basic **RFM values**:

```sql
SELECT 
    CUSTOMERNAME,
    MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y')) AS LAST_ORDER_DATE,
    COUNT(DISTINCT ORDERNUMBER) AS F_VALUE,
    ROUND(SUM(SALES),0) AS M_VALUE
FROM SAMPLE_SALES_DATA
GROUP BY CUSTOMERNAME;
```

- **Recency (R_VALUE)**: Days since last purchase.
- **Frequency (F_VALUE)**: Number of distinct orders.
- **Monetary (M_VALUE)**: Total sales value.

Example using a subquery for filtering customers by recency:

```sql
SELECT * FROM
(
    SELECT 
        CUSTOMERNAME,
        DATEDIFF(
            (SELECT MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y')) FROM SAMPLE_SALES_DATA), 
            MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y'))
        ) AS R_VALUE,
        COUNT(DISTINCT ORDERNUMBER) AS F_VALUE,
        ROUND(SUM(SALES),0) AS M_VALUE
    FROM SAMPLE_SALES_DATA
    GROUP BY CUSTOMERNAME
) AS SUMMARY_TABLE
WHERE R_VALUE BETWEEN 50 AND 100;
```

---

## **4. RFM Score Calculation and Segmentation**

Create a **view `RFM`** combining R, F, M values and scoring:

```sql
CREATE OR REPLACE VIEW RFM AS
WITH CUSTOMER_SUMMARY_TABLE AS
(
    SELECT 
        CUSTOMERNAME,
        DATEDIFF(
            (SELECT MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y')) FROM SAMPLE_SALES_DATA), 
            MAX(STR_TO_DATE(ORDERDATE, '%d/%m/%y'))
        ) AS RECENCY_VALUE,
        COUNT(DISTINCT ORDERNUMBER) AS FREQUENCY_VALUE,
        ROUND(SUM(SALES),0) AS MONETARY_VALUE
    FROM SAMPLE_SALES_DATA
    GROUP BY CUSTOMERNAME
),
RFM_SCORE AS
(
    SELECT 
        S.*,
        NTILE(5) OVER(ORDER BY RECENCY_VALUE DESC) AS R_SCORE,
        NTILE(5) OVER(ORDER BY FREQUENCY_VALUE ASC) AS F_SCORE,
        NTILE(5) OVER(ORDER BY MONETARY_VALUE ASC) AS M_SCORE
    FROM CUSTOMER_SUMMARY_TABLE AS S
),
RFM_COMBINATION_SCORE AS
(
    SELECT 
        R.*,
        (R_SCORE + F_SCORE + M_SCORE) AS TOTAL_RFM_SCORE,
        CONCAT_WS('', R_SCORE, F_SCORE, M_SCORE) AS RFM_SCORE_COMBINATION
    FROM RFM_SCORE AS R
)
SELECT
    RC.*,
    CASE
        WHEN RFM_SCORE_COMBINATION IN (455, 542, 544, 552, 553, 452, 545, 554, 555) THEN 'Champions'
        WHEN RFM_SCORE_COMBINATION IN (344, 345, 353, 354, 355, 443, 451, 342, 351, 352, 441, 442, 444, 445, 453, 454, 541, 543, 515, 551) THEN 'Loyal Customers'
        WHEN RFM_SCORE_COMBINATION IN (513, 413, 511, 411, 512, 341, 412, 343, 514) THEN 'Potential Loyalists'
        WHEN RFM_SCORE_COMBINATION IN (414, 415, 214, 211, 212, 213, 241, 251, 312, 314, 311, 313, 315, 243, 245, 252, 253, 255, 242, 244, 254) THEN 'Promising Customers'
        WHEN RFM_SCORE_COMBINATION IN (141, 142,143,144,151,152,155,145,153,154,215) THEN 'Needs Attention'
        WHEN RFM_SCORE_COMBINATION IN (113, 111, 112, 114, 115) THEN 'About to Sleep'
        ELSE 'OTHER'
    END AS CUSTOMER_SEGMENT
FROM RFM_COMBINATION_SCORE AS RC;
```

---

## **5. Segment-Wise Aggregation**

### Total and average monetary value per segment:

```sql
SELECT 
    CUSTOMER_SEGMENT,
    ROUND(SUM(MONETARY_VALUE), 0) AS TOTAL_MONETARY_VALUE,
    ROUND(AVG(MONETARY_VALUE), 0) AS AVERAGE_MONETARY_VALUE
FROM RFM
GROUP BY CUSTOMER_SEGMENT
ORDER BY TOTAL_MONETARY_VALUE DESC;
```

### Total sales and quantity per segment:

```sql
SELECT 
    CUSTOMER_SEGMENT,
    SUM(QUANTITYORDERED) AS TOTAL_QUANTITY_ORDERED,
    ROUND(SUM(SALES),0) AS TOTAL_SALES_AMOUNT
FROM SAMPLE_SALES_DATA AS S
LEFT JOIN RFM AS R ON S.CUSTOMERNAME = R.CUSTOMERNAME
GROUP BY CUSTOMER_SEGMENT
ORDER BY TOTAL_SALES_AMOUNT DESC;
```

---

## **6. Notes**

- `STR_TO_DATE(ORDERDATE, '%d/%m/%y')` is used to convert order dates from string to date.  
- `NTILE(5)` is used to create quintile-based RFM scores.  
- Customer segmentation is based on RFM score combinations.  
- Useful for **targeted marketing, loyalty programs, and customer retention strategies**.

---

## **7. Sample Preview Queries**

```sql
SELECT * FROM SAMPLE_SALES_DATA LIMIT 10;
SELECT * FROM RFM LIMIT 10;
```

---

## **8. References**

- RFM Analysis concepts: *Recency, Frequency, Monetary value segmentation*  
- MySQL functions: `DATEDIFF()`, `NTILE()`, `STR_TO_DATE()`, `CONCAT_WS()`, `ROUND()`
