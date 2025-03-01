# Fetch-TakeHome
Take home test for Fetch Senior Data Analyst Position Feb 2025. 

## Part 1: Exploration

There are several columns across all three tables that have missing values. More importantly, there are a high number of barcodes missing from the Transactions table and Category_3 assingments from the Products table. The Category assingments can become an issue when a more granular analysis is needed, and the category depth level can become an issue. 

```sql
-- Checking for any missing values in the transactions table
SELECT
	COUNT(*) AS total_rows,
	SUM(CASE WHEN RECEIPT_ID IS NULL OR RECEIPT_ID = " " THEN 1 ELSE 0 END) AS missing_receipt_id,
	SUM(CASE WHEN PURCHASE_DATE IS NULL OR PURCHASE_DATE = " " THEN 1 ELSE 0 END) AS missing_purchase_date,
	SUM(CASE WHEN FINAL_SALE IS NULL OR FINAL_SALE = " " THEN 1 ELSE 0 END) AS missing_final_sale,
	SUM(CASE WHEN USER_ID IS NULL OR USER_ID = " " THEN 1 ELSE 0 END) AS missing_user_id
FROM transactions t;

-- Checking for any missing values in the products table
SELECT 
    COUNT(*) AS total_rows,
    SUM(CASE WHEN BARCODE IS NULL OR BARCODE = " " THEN 1 ELSE 0 END) AS missing_barcode,
    SUM(CASE WHEN BRAND IS NULL OR BRAND = '' THEN 1 ELSE 0 END) AS missing_brand,
    SUM(CASE WHEN CATEGORY_2 IS NULL OR CATEGORY_2 = '' THEN 1 ELSE 0 END) AS missing_category
FROM products p;
```
Python Visual:
![Screenshot 2025-03-01 at 2 18 31 PM](https://github.com/user-attachments/assets/ee6e326e-1d92-442d-94d7-1429a2cd43ac)

The barcodes missing leads to inaccurate data that would exclude valid transactions from the dataset. This becomes an issue when there is a high level of mismatched barcodes between Transactions table and the Products table. This excludes valid transactions with barcodes because of an outdated Products table. 

```sql
SELECT COUNT(DISTINCT t.BARCODE)
FROM transactions t
LEFT JOIN products p ON t.BARCODE = p.BARCODE --finding any mismatches on the transactions table
WHERE p.BARCODE IS NULL; -- pull out the mismatches
```

Python Visual:
![Screenshot 2025-03-01 at 2 20 17 PM](https://github.com/user-attachments/assets/2aa41a85-a96a-4350-a40d-e7111f649672)

## Part 2: SQL Queries
### What are the top 5 brands by receipts scanned among users 21 and over?
```sql
Select p.BRAND, COUNT(DISTINCT t.RECEIPT_ID) AS total_receipts_scanned
FROM transactions t
JOIN users u ON t.USER_ID = u.ID -- joining both datasets to identify who made purchases & is 21+
JOIN products p ON t.BARCODE = p.BARCODE --joining both datasets to determine top brands
WHERE u.BIRTH_DATE  <= date('now', '-21 years') -- calculating the age
AND TRIM(p.Brand) <> "" -- exclude any empty brand fields
GROUP BY p.BRAND 
ORDER BY total_receipts_scanned DESC
LIMIT 5;
```
Assuming that the scans happened when the user was 21 or over, the top brands are Nerds Candy, Dove, Sour Patch Kids, Hershey's & Coca Cola in order. 

### Who are Fetch’s power users?
```sql
SELECT USER_ID,
       COUNT(RECEIPT_ID) AS receipt_count, -- total number of receipts by user
       COUNT(DISTINCT PURCHASE_DATE) AS active_days, -- total distinct days active by user
       (COUNT(RECEIPT_ID) * 0.7 + COUNT(DISTINCT PURCHASE_DATE) * 0.3) AS power_score
-- weighing receipt volume more than active days to generate a score per user
FROM transactions t 
GROUP BY USER_ID
ORDER BY power_score DESC
LIMIT 10;
```
Assuming that a power user is defined by the following: 
1. Uploads multiple receipts, ideally all transactions they've done.
2. Uploads consistently across multiple days, ideally every time they do a shopping trip.
3. Volume of receipts is weighted more favorably compared to number of days uploading receipts.

Operating under these assumptions we get that User 64e62de5ca929250373e6cf5 is the top user with 22 receipts uploaded across 8 days. Using the calculation above, nets a "Power Score" of 17.8. 

### Which is the leading brand in the Dips & Salsa category?
```sql
SELECT p.BRAND, SUM(t.FINAL_QUANTITY) AS total_units_sold
FROM products p
JOIN transactions t ON t.BARCODE = p.BARCODE 
WHERE p.CATEGORY_2 = "Dips & Salsa"
AND TRIM(p.Brand) <> ""
GROUP BY p.BRAND
ORDER BY total_units_sold DESC -- evaluating a leading brand based on the # of units sold
LIMIT 5;
```
Assuming: 
1. A category leader would have higher velocity of products, here being measured in volume of receipts scanned
2. Each barcode is a unique identifier for a SKU for that brand, i.e. a 4pk differentiated from a 10pk, and flavor options differentiated as well.

With this we observe that Tostitos is the leading brand in terms of volume of units sold by receipts scanned that had barcodes coded back to Tostitos brand. 

## Part 3: Potential Slack / Email Message: 

Hello, 

I wanted to provide an update on the recent investigation I conducted on the datasets. 
  1. There are nearly 20% of barcode data missing from the Transactions dataset, which can cause issues when cross-referencing with other data sources. Additionally, some barcodes in the Transactions dataset do not match with the Products dataset, which can exclude valid transactions from the final analysis and skew the results.
  2. A key insight I've uncovered is that among the Dips & Salsa category, Tostitos leads in units sold by 25% compared to the second place brand, Fritos. For reference, the next top 3 brands after Tostitos are at most 0.2% apart in units sold. That's quite a margin! 
  3. To resolve the data quality issues, an updated Transactions dataset would be needed to fill in the gaps on the barcode data. Also, an updated Products dataset with more barcode data filled in can ensure a higher quality analysis with more data match.

Please let me know if you can assist in providing these updated datasets or if you have any suggestions for resolving these issues. 

Thank you!

Jaime
