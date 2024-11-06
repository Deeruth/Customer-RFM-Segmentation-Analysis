# Customer-RFM-Segmentation-Analysis
This project performs RFM (Recency, Frequency, Monetary) segmentation analysis on customer data to categorize customers into various segments such as Champions, Loyalists, Potential Loyalists, Promising, Need Attention, At Risk, and Inactive

# Data Source
The analysis uses the following tables:

Order Table: Contains details of individual customer orders.
Order Group Table: Aggregates orders by customer.
Store Table: Information on store locations and identifiers.
Agent Table: Sales agent information related to customer transactions.
Regions Table: Geographic region information for customer segmentation.

## Methodology

# RFM Scoring
Each customer was scored based on:

Recency (R): How recently the customer made a purchase.
Frequency (F): How often the customer made purchases.
Monetary Value (M): The total spend by the customer.

# MySQL Queries for RFM Calculation

1. Recency Calculation: Determine the most recent order date per customer.

``SELECT customer_id,
       DATEDIFF(CURRENT_DATE, MAX(order_date)) AS Recency
FROM orders
GROUP BY customer_id;``

2. Frequency Calculation: Count the total orders per customer

``SELECT customer_id,
       COUNT(order_id) AS Frequency
FROM orders
GROUP BY customer_id;``

3. Monetary Value Calculation: Sum the total spend by each customer.

``SELECT customer_id,
       SUM(total_amount) AS Monetary
FROM orders
GROUP BY customer_id;``

After obtaining these scores, I used Pecentile-Based Binning to standardized them into RFM groups with MySQL, creating tiered RFM scores that allowed us to define customer segments.

1. Recency (lower is better):

``WITH ranked_recency AS (
    SELECT customer_id,
           Recency,
           NTILE(10) OVER (ORDER BY Recency ASC) AS Recency_Tile
    FROM (
        SELECT customer_id,
               DATEDIFF(CURRENT_DATE, MAX(order_date)) AS Recency
        FROM orders
        GROUP BY customer_id
    ) AS recency_data
)
SELECT customer_id,
       Recency,
       CASE
           WHEN Recency_Tile <= 3 THEN 'High'
           WHEN Recency_Tile BETWEEN 4 AND 7 THEN 'Medium'
           ELSE 'Low'
       END AS Recency_Category
FROM ranked_recency;``

2. Frequency (higher is better):

``WITH ranked_frequency AS (
    SELECT customer_id,
           Frequency,
           NTILE(10) OVER (ORDER BY Frequency DESC) AS Frequency_Tile
    FROM (
        SELECT customer_id,
               COUNT(order_id) AS Frequency
        FROM orders
        GROUP BY customer_id
    ) AS frequency_data
)
SELECT customer_id,
       Frequency,
       CASE
           WHEN Frequency_Tile <= 3 THEN 'High'
           WHEN Frequency_Tile BETWEEN 4 AND 7 THEN 'Medium'
           ELSE 'Low'
       END AS Frequency_Category
FROM ranked_frequency;``

3. Monetary (higher is better):

``WITH ranked_monetary AS (
    SELECT customer_id,
           Monetary,
           NTILE(10) OVER (ORDER BY Monetary DESC) AS Monetary_Tile
    FROM (
        SELECT customer_id,
               SUM(total_amount) AS Monetary
        FROM orders
        GROUP BY customer_id
    ) AS monetary_data
)
SELECT customer_id,
       Monetary,
       CASE
           WHEN Monetary_Tile <= 3 THEN 'High'
           WHEN Monetary_Tile BETWEEN 4 AND 7 THEN 'Medium'
           ELSE 'Low'
       END AS Monetary_Category
FROM ranked_monetary;``

# RFM Segmentation in MySQL
I grouped customers based on these scores into the following segments:

Champions: High recency, frequency, and monetary scores.
Loyalists: High frequency and monetary scores with a recent purchase.
Potential Loyalists: High frequency but lower recent activity.
Promising: Moderate scores across metrics.
Need Attention: Average spenders with declining engagement.
At Risk: High spenders with low recent activity.
Inactive: Customers who have not engaged recently.

## PowerBI Dax for RFM Calculation
1. Calculated Recency by measuring the number of days between the last purchase date and today’s date

``Recency = DATEDIFF('Cust RFM'[Last_PurchaseDate], TODAY(), DAY)``

2. Calculated Monetary value from inception

``Monetary = CALCULATE( 'Sales Measures'[Sales], FILTER(order_groups, order_groups[store_id] = 'Cust RFM'[store_id]))``

3. Calculated orders received value from inception

``Frequency = CALCULATE( 'Orders Measures'[Orders Fulfilled], FILTER(order_groups, order_groups[store_id] = 'Cust RFM'[store_id]))``


After obtaining these scores, I standardized them into RFM groups using DAX, creating tiered RFM scores that allowed us to define customer segments.

# Tiered RFM Scores Dax in Power BI

Used RANKX to rank customers based on Recency in ascending order (where lower recency days get higher ranks).
The division by the maximum rank, divided by 5, normalizes the R score into a range, likely from 1 to 5, by grouping ranks into quintiles.

``R score = RANKX(
    ALL('Cust RFM'), 'Cust RFM'[Recency], , ASC
) /
DIVIDE(MAXX(ALL('Cust RFM'), RANKX(ALL('Cust RFM'), 'Cust RFM'[Recency])), 5)``

Used RANKX to rank customers based on Monetary in ascending order.

``M score = 
RANKX(
    ALL('Cust RFM'),'Cust RFM'[Monetary],,DESC)
    /
    DIVIDE(MAXX(ALL('Cust RFM'), RANKX(ALL('Cust RFM'), 'Cust RFM'[Monetary])),5)``

Used RANKX to rank customers based on Frequency in ascending order

``F score = 
RANKX(
    ALL('Cust RFM'),'Cust RFM'[Orders Count],,DESC)
    /
    DIVIDE(MAXX(ALL('Cust RFM'), RANKX(ALL('Cust RFM'), 'Cust RFM'[Orders Count])),5)``

Calculating RFM Score- The overall RFM Score is the sum of the individual R, F, and M scores. This composite score is crucial for segmenting customers.
    ``RFM Score = 'Cust RFM'[R score] + 'Cust RFM'[F score] + 'Cust RFM'[M score]``
    
# RFM segmentation Dax in Power BI
I used DAX to calculate RFM scores, grouping customers based on these scores.
This SWITCH function assigns customer segments based on the RFM Score:
Champions (RFM Score ≤ 1.99): Most engaged and valuable customers.
Loyal Customers through Inactive: The scores increase progressively, with Inactive being the least engaged group.
Each range threshold (1.99, 4.00, etc.) aligns customers into progressively less engaged or valuable segments.

``RFM Segment = 
SWITCH(
    TRUE(),
    'Cust RFM'[RFM Score] <= 1.99, "Champions",
    'Cust RFM'[RFM Score] <= 4.00, "Loyal Customers",
    'Cust RFM'[RFM Score] <= 6.00, "Potential Loyalist",
    'Cust RFM'[RFM Score] <= 8.00, "Promising",
    'Cust RFM'[RFM Score] <= 10.00, "Need Attention",
    'Cust RFM'[RFM Score] <= 12.00, "At Risk",
    'Cust RFM'[RFM Score] <= 14.00, "Lost Customers",
    'Cust RFM'[RFM Score] > 14.00, "Inactive",
    "Unclassified"
)``


# Data Visualization in Power BI
Using Power BI, I created visuals to represent the segmentation distribution, trends, and insights across customer demographics, purchase habits, and regions.

![image](https://github.com/user-attachments/assets/23e7c70e-d791-456a-9dfb-ae89b9b088d0)
![image](https://github.com/user-attachments/assets/522b6da9-6ace-462b-a579-980ccc588f50)
![image](https://github.com/user-attachments/assets/14fdfd6e-d123-4da0-af29-a30ec8c94812)



# Insights and Recommendations

Customer Service: Focus on re-engaging customers in the "Need Attention" and "At Risk" segments.
Sales Team: Prioritize interactions with "Champions" and "Loyalists" to maximize revenue from high-value customers.
Marketing: Tailor campaigns for "Promising" and "Potential Loyalists" to nurture loyalty and encourage repeated engagement.
