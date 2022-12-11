# <center>CHURN COHORT ANALYSIS
## Using a sample data of 541,909 records

## About The Dataset
### Context
This is a transnational dataset, which contains all the transactions occurring between 01/12/2010 and 09/12/2011 for a UK-based online retail business. 

The company mainly sells unique all-occasion gifts. 

Many customers of the company are wholesalers.

## Data Source:
The data was downloaded from University of California Irvine (UCI) [website](https://archive.ics.uci.edu/ml/datasets/Online+Retail), curtesy of Dr. Daqing Chen, Director of Public Analytics group. chend '@' lsbu.ac.uk, School of Engineering, London South Bank University, London SE1 0AA, UK.

The project based on guide by Angelina Frimpong’s YouTube teaching with link [here](https://www.youtube.com/watch?v=LXqpx9mr0Is).

## Data Attributes:
*	InvoiceNo: Invoice number. Nominal, a 6-digit integral number uniquely assigned to each transaction. If this code starts with letter 'c', it indicates a cancellation.
*	StockCode: Product (item) code. Nominal, a 5-digit integral number uniquely assigned to each distinct product.
*	Description: Product (item) name. Nominal.
*	Quantity: The quantities of each product (item) per transaction. Numeric.
*	InvoiceDate: Invoice Date and time. Numeric, the day and time when each transaction was generated.
*	UnitPrice: Unit price. Numeric, Product price per unit in sterling.
*	CustomerID: Customer number. Nominal, a 5-digit integral number uniquely assigned to each customer.
*	Country: Country name. Nominal, the name of the country where each customer resides.

The dataset was an Excel file, which I converted to CSV for easy manipulation using data tools. There were a total of 541,909 records.
 
Here is a peep look at the dataset structure:

![image](https://user-images.githubusercontent.com/69392408/206926205-2b2dd589-f3a8-465a-98c5-2aeb5cef81c7.png)

## Objective Of The Analysis
The main objective of this churn analysis project is to evaluate a company's customer churn (loss) rate in order to reduce it.

### What is App Churn Rate?
Churn rate, also known as the rate of attrition, is the percentage of users who stopped patronizing a company within a given period.

### Why Churn Rate Matters
Churn rate suppresses growth. It’s the leaky bucket analogy: as customers drop off, the company will keep struggling to refill its bucket by adding new (acquisition) customers.  Facts to note:
*	Acquiring a new customer is 5 to 25 times more expensive than retaining one;
*	Reducing churn by just 5% can boost profitability by 75%;
*	Improving retention has about 4 times greater impact on growth than acquisition;
*	The probability of selling to an existing customer is 60 to 70%, but only an average of 15% for a new prospect.

This is why it is important to be mindful of churn rate and retentions rate.

## Data Analytic Tool
For this project, SQL - CTE (via BigQuery platform) and Tableau were used.

## Data Preparations
The downloaded dataset named “online_retail.xlsx” was first converted to a CSV file called “online_retail.csv”.

The csv file was imported into Google cloud-based BigQuery, where SQL was used inspect and clean the dataset. The following are some of the cleaning done:
*	Checking the records, 135, 080 were found without CustomerID, which is very vital, and 406,829 records had the needed CustomerID. These records with no CustomerID were removed.
*	Also, out of these 406,829 records, 397,884 records had Quantity and Unit Price, which were also needed.
*	Finally, some records were duplicated. These were a total 5, 215 records and there were removed, leaving us with 392, 669 clean records.

The SQL codes used to clean these records are:

```
#####CLEAN RECORD
WITH retails AS
(
  -- First we get the records that has CustomerID
 SELECT InvoiceNo
      ,StockCode
      ,Description
      ,Quantity
      ,InvoiceDate
      ,UnitPrice
      ,CustomerID
      ,Country
 FROM `churn-370413.online_retail.retail`
 WHERE CustomerID != 0 
),
wtQnty_n_price AS 
(
  --Let us get records that have customerID, Quantity and Unit Price 
  SELECT *
  FROM retails
  WHERE Quantity > 0 AND UnitPrice > 0
),
check_duplicate AS
(
  -- Let us check and remove the duplicated records
 SELECT * , ROW_NUMBER() OVER (PARTITION BY InvoiceNo, StockCode, Quantity    
 ORDER BY InvoiceDate)dup_flag
 FROM wtQnty_n_price
)

SELECT * 
FROM check_duplicate
WHERE dup_flag = 1
```

## Data Processing & Analysis

### Generating Cohort Values

***"Cohort analysis is an analytical technique that categorizes and divides data into groups with common characteristics prior to analysis."***

We need 4 main values to successfully generate cohort analysis values. These are:
1.	Unique Identifier (CustomerID)
2.	Initial Start Date (First Invoice Date)
3.	Revenue Data
4.	Cohort Index

Out of these, we have the CustomerID and Revenue Data in place. Using SQL we need derive the remaining 2 values. 

We need the cohort index, this is the number of months since the first transaction for each customer.

```
#####BEGIN COHORT ANALYSIS
 
SELECT
	CustomerID,
	min(InvoiceDate) first_purchase_date,
	DATEFROMPARTS(year(min(InvoiceDate)), month(min(InvoiceDate)), 1) Cohort_Date
INTO #cohort_table
FROM #clean_records
GROUP BY CustomerID
```

```
####Create Cohort Index
SELECT
  mmm.*,
  (year_diff * 12 + month_diff + 1) AS cohort_index
FROM
  (
    SELECT
      mm.*,
      (invoice_year - cohort_year) AS year_diff,
      (invoice_month - cohort_month) AS month_diff , 
    FROM
      (
        SELECT
          m.*,
          c.Cohort_Date,
          extract(YEAR FROM date(m.InvoiceDate)) invoice_year,
          extract(MONTH FROM date(m.InvoiceDate)) invoice_month,
          extract(YEAR FROM date(c.Cohort_Date)) cohort_year,
          extract(MONTH FROM date(c.Cohort_Date)) cohort_month
        FROM `churn-370413.online_retail.clean_record` m
        LEFT JOIN `churn-370413.online_retail.cohort_table` c
          ON m.CustomerID = c.CustomerID
      )mm
  )mmm

```

Here is the output:

![image](https://user-images.githubusercontent.com/69392408/206926500-56f7b7ec-b026-4251-b62f-3b0d4a3b8fc1.png)


Our Cohort Index shows a range from 1 to 13.
Next we need to generate Cohort Pivot table to show retentions.


#### PIVOT TABLE
```
###Pivot Data to see the cohort table
SELECT  *
FROM (
  SELECT DISTINCT 
    CustomerID,
    Cohort_Date,
    cohort_index
  FROM `churn-370413.online_retail.cohort_index`
)tbl
PIVOT(
  Count(CustomerID)
  FOR Cohort_Index IN (1,2,3,4,5,6,7,8,9,10,11,12,13) 
) AS pivot_table;
```


SQL Output:

![image](https://user-images.githubusercontent.com/69392408/206926533-32f16d94-626f-4833-8ee0-14b2596b2e2e.png)

Tableau Output:

![image](https://user-images.githubusercontent.com/69392408/206926545-728721d1-70ab-4dd4-86cb-de6b80b2f76a.png)
 

### PIVOT RATE
```
-- We will convert the output of the above to Percentages:
-- (each ROW Value divided by First Row Value multiplied by 100)

SELECT Cohort_Date,
    ROUND((_1/_1 * 100),2) as i1, 
    ROUND((_2/_1 * 100),2) as i2, 
    ROUND((_3/_1 * 100),2) as i3,  
    ROUND((_4/_1 * 100),2) as i4,  
    ROUND((_5/_1 * 100),2) as i5, 
    ROUND((_6/_1 * 100),2) as i6, 
    ROUND((_7/_1 * 100),2) as i7, 
    ROUND((_8/_1 * 100),2) as i8, 
    ROUND((_9/_1 * 100),2) as i9, 
    ROUND((_10/_1 * 100),2) as i10,   
    ROUND((_11/_1 * 100),2) as i11,  
    ROUND((_12/_1 * 100),2) as i12,  
    ROUND((_13/_1 * 100),2) as i13
FROM `churn-370413.online_retail.pivot`
```

SQL Output:

![image](https://user-images.githubusercontent.com/69392408/206926579-49035cba-cd81-467d-bf8d-175afd884350.png)


Tableau Output:

![image](https://user-images.githubusercontent.com/69392408/206926581-fb3c3f0f-5acf-4c63-814b-bbe9e3be3499.png)


### Generating Churn Values
Finally, we need to generate values that will enable us calculate the churn rate.

```
  SELECT
    CustomerID,
    min(InvoiceDate) first_purchase_date,
    max(InvoiceDate) last_purchase_date,
    EXTRACT(MONTH FROM (max(InvoiceDate))) invoice_month,
    DATETIME_DIFF(DATETIME(TIMESTAMP(max(InvoiceDate))), DATETIME(TIMESTAMP(min(InvoiceDate))), MONTH) as months_active
  FROM clean_record
  GROUP BY CustomerID
  ORDER BY max(InvoiceDate) DESC
```

SQL output:

![image](https://user-images.githubusercontent.com/69392408/206926608-a00faddf-5014-4102-9c82-dc62ad840a32.png)


Tableau Output:

![image](https://user-images.githubusercontent.com/69392408/206926618-1e712e87-6bf8-47f6-965b-154de9e0c3b5.png)


## Data Visualization Dashboard

![image](https://user-images.githubusercontent.com/69392408/206926638-3d0beee9-a06f-4865-be97-f9f3f14c8948.png)


## Conclusion

As we can see, the churn rate is 40.99%. It is high, but the most important thing is to see how to gain them back and reduce these rate. This can be done by first finding out why patronages stop.
