# Airline Efficiency and Profitability Analysis 


## Project Overview

This analysis project explored and revealed regions that has the high performing airlines in terms of efficiency and profit and what are the major drivers of these performance metrics.


## Data Sources
Airline_Data: The dataset for this project is an "airline_data.csv" file that contains 14 fields and 100 records of airlines across 6 regions.



## Airline_Data Dictionary

| Column Name            | Description                                                                 |
|------------------------|-----------------------------------------------------------------------------|
| `iata_code`            | The unique 2- or 3-letter airline code assigned by IATA                    |
| `airline_name`         | Full name of the airline                                                    |
| `region`               | Geographical region where the airline is primarily based or operates       |
| `functional_currency`  | The currency used in the airline financial reporting                        |
| `ebit_usd`             | Earnings Before Interest and Taxes (EBIT) in USD – a measure of profitability |
| `load_factor`          | Percentage of seating capacity filled with passengers (efficiency metric)  |
| `low_cost_carrier`     | Indicates whether the airline is a low-cost carrier ('Y' or 'N')           |
| `airline_age`          | Age of the airline in years                                                 |
| `num_routes`           | Number of flight routes operated by the airline                             |
| `passenger_yield`      | Revenue per revenue passenger kilometer (revenue per passenger distance)    |
| `ask`                  | Available Seat Kilometers (seats × distance flown) – capacity measure       |
| `avg_fleet_age`        | Average age of aircraft in the airline’s fleet (years)                      |
| `fleet_size`           | Total number of aircraft in the airline’s fleet                             |
| `aircraft_utilisation` | Avg hours each aircraft operates per day                                    |


## Tools
- Excel : Data normalization
- SQL Server : Data cleaning, manipulation and analysis


## Data Cleaning/Preparation
- Data loading and inspection
- Handling missing values
- Segmenting data for analysis


## Exploratory Data Analysis
- What is the total number of airlines in each region?
- How many are actively operating and how many do not?
- What is total number of low_cost_carriers and full_service_carriers in each region?


## Data Analysis

During the exploratory analysis, 34 records of 100 where discovered not having values for columns like ask(Available Seat Kilometers) which is an important values to track efficiency of airlines

```sql

SELECT *
FROM airline_data
WHERE ask IS NULL
```
Other data quality issues faced includes:
 - airlines without ask value but a loadfactor(percentage of the ask available that were occupied by passengers) 
 - airlines that provided the revenue(ebit_usd) values only.
These aforementioned problems led me to segmenting the airlines according to thier provided values and also did feature engineering on low_cost_carrier column.


```sql
CREATE VIEW AirlineData AS
SELECT 
		iata_code,
		airline_name,
		region,
		ebit_usd,
		load_factor,
		passenger_yield,
		ask,
		CASE WHEN low_cost_carrier = 'Y' THEN 'low_cost_carrier' ELSE 'full_service_carrier' END AS carrier_type,
		CASE WHEN ask IS NOT NULL AND load_factor IS NOT NULL THEN 'operated'
			 WHEN load_factor IS NOT NULL  THEN 'operated'
			 WHEN ask IS NOT NULL THEN 'active'
			 WHEN ebit_usd IS NOT NULL THEN 'no_efficiency_data'
			 ELSE 'inactive'
		END AS airline_status
   FROM airline_data
```
For reuseability of the features i engineered in the dataset, i created a view that houses my prepared data. In the view,  i picked only columns that are important for my analysis and added new ones(carrier_type and airline_status). 

-The carrier_type classified the airlines into low-cost carriers and full-service carriers.

The airline_status classified airlines based on their values/features provided,the condition for classifying them are as follows:
- Airlines with a ask and load factor value means the airline operated therefore clasified as "operated"
- Airlines with with only load factor as also classified "operator" meaning they operated but the ask was not provided
- Airlines with ask value only are classified "active" which means they have availabe seats but not patronized by passengers
- Airlines with ebit-usd value are classified as "no_efficiency_data"  while airlines that didn't fail in any of these bins/categories are tagged 'inactive'

#### Note: For this analysis, I focused exclusively on airlines tagged as "operated" only, as they are the most relevant for evaluating both efficiency and profitability


  ```sql
     SELECT 
	   ISNULL(region,'Total') AS region,
	   COUNT(iata_code) AS total_airline,
	   COUNT(CASE WHEN airline_status IN('operated','active') THEN 1 END) AS actives,
	   COUNT(CASE WHEN airline_status IN('no_efficiency_data','inactive') THEN 1 END) AS not_actives,
	   COUNT(CASE WHEN carrier_type = 'low_cost_carrier' THEN 1 END) AS n_LCC,
	   COUNT(CASE WHEN carrier_type = 'full_service_carrier' THEN 1 END) AS n_FSC
  FROM AirlineData
  GROUP BY ROLLUP(region)
   ```

#### Query output

<img width="420" height="170" alt="eda_airline" src="https://github.com/user-attachments/assets/72458b64-2dfb-430b-ba7d-a8e79b0893bb" />

The query reveals the number of airlines operating in each region, including the total count, the number of active and inactive airlines, as well as a breakdown of low-cost and full-service carriers per region.




### Thoughts of handling question

Airlines without ASK which is the primary metric for measuring efficiency and load factor were excluded from the analysis. However, airlines that had a load factor but no ASK value were still considered, based on the assumption that the ASK data was simply unavailable.The analysis followed a logical progression: it prioritized efficiency metrics before moving on to profitability, as operational efficiency is a key foundation for understanding airline performance.
```sql

 -- Airline Efficiency_metrics

WITH 
air_efficiency
AS(SELECT region,
	   AVG(load_factor)*100 AS avg_loadfactor,
	   AVG(passenger_yield) AS avg_passenger_yield
FROM AirlineData
WHERE airline_status = 'operated'
GROUP BY  region
),

---filtered out airlines running at loss
ranking
AS(SELECT a.region,
	   e.avg_loadfactor,
	   avg_passenger_yield,
	   AVG(ebit_usd) AS revenue,
	   DENSE_RANK() OVER(ORDER BY e.avg_loadfactor DESC,avg_passenger_yield DESC,AVG(ebit_usd) DESC) AS rank
FROM AirlineData  a
JOIN air_efficiency e
ON a.region = e.region
WHERE airline_status = 'operated' AND ebit_usd > 0
GROUP BY  a.region,e.avg_loadfactor,avg_passenger_yield
)

SELECT *
FROM ranking 
WHERE rank <=3

```

### Query output - Top 3 regions 

The query reveals that North,Latin America and Europe leads in efficiency and profit. Eventhough Africa has the highest passenger yield of 0.11 per KM, It falls way lower in loadfactor compared to other regions.


<img width="441" height="100" alt="top 3 regions" src="https://github.com/user-attachments/assets/24c9b1f1-a95f-4442-90a2-539347430878" />


### A deep dive into the Top 3 regions

```sql

SELECT region,
	   carrier_type, 
	   COUNT(CASE WHEN airline_status = 'operated' THEN 1 END) AS actives,
	   AVG(load_factor)*100 AS avg_loadfactor,
	   AVG(ebit_usd) AS revenue
FROM AirlineData 
WHERE airline_status = 'operated' AND region IN('North America','Latin America','Europe')
GROUP BY region,carrier_type
ORDER BY region,avg_loadfactor DESC
```

<img width="441" height="132" alt="low vs full_service carriers" src="https://github.com/user-attachments/assets/b41d01d2-abf4-401d-80a7-fe9711f9fab2" />
