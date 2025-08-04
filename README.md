# Airline Efficiency and Profitability Analysis 


## Project Overview

This analysis project explored and revealed regions that has the high performing airlines in terms of efficiency and profit and what are the major drivers of these performance metrics.


## Data Sources
Airline_Data: The dataset for this project is an "airline_data.csv" file that contains 14 fields and 100 records of airlines across 6 regions.


##  Airline_Data – Data Dictionary

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
Other data quality issues faced include airlines without ask having a loadfactor(percentage of the ask available that were occupied by passengers) and records that provided the revenue(ebit_usd) values only. These aforementioned problems led me to segmenting the airlines according thier provided features and also i also did feature engineering on low_cost_carrier column.


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
For reuseability of the features i engineered in the dataset, i created a view that houses my prepared data. In the view,  i picked only columns that are important for my analysis and added new ones carrier_type and airline_status. 

-The carrier_type classified the airlines into low-cost carriers and full-service carriers.

-The airline_status classified airlines based on their values/features provided,the condition for classifying them are as follows:
- Airlines with a ask and load factor value means the airline operated therefore clasified as "operated"
- Airlines with with only load factor as also classified "operator" meaning they operated but the ask was not provided
- Airlines with ask value only are classified "active" which means they have availabe seats but not patronized by passengers
- Airlines with ebit-usd value are classified as "no_efficiency_data"  while airlines that didn't fail in any of these bins/categories are tagged 'inactive'
- N/B: For my analysis, i focused on the airlines tagged operated only, as they are those that can't help answers of both efficiency and profit.



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
![image alt](https://github.com/SQLDrivenAbdul/Airline-Regional-Analysis/raw/8abfcdc360fcd6704199d1137915141878d22c36/ailine_eda.PNG)
