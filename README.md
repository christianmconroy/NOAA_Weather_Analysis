#  Exploing NOAA ISD Weather Data and Predicting Wildfire Acreage from Weather Patterns 

The purpose of this project is to predict wildfire spread based on monthly weather patterns. 

## Executive Summary

- 10,200 tenants, with 1,254 holding more than one contract\*
- Majority of tenants (6,622) have contract history of at least 24 months
- 86.7% deposits returned while 5.1% partially returned and 8.2% forfeited
- 32 tenants appeared likely to have been evicted based on gaps in amount of expected rent and number of expected payments
- Almost all likely evictions had pattern of late rent payment and three-quarters paid the highest late fee on at least one occasion
- Random Forest on an upsampled version of the data offered the best performance as measured by AUC
- Contract history, age at the start of contract history, and use of cash for payment appeared to be most important in predicting bad tenants

*\*Does not include transactions without contract data*

## The Data<br><sup> Understanding tenant and contract history</sup>

In order to develop the initial proof of concept for model development without the use of cloud-based storage or compute solutions, not all available data was used in model training. Instead, model development focused only
on weather pattern and wildfire data for the U.S. state of California aggregated up to the monthly level. The weather data came from NOAA's 
Integrated Surface Data (ISD) Lite,  a global database that consists of hourly and synoptic surface observations compiled from numerous sources into a 
single common ASCII format and common data model. Fields included: 

| Weather Variable                          | Format                          |
| ------------------------------------------|:-------------------------------:|
| Air temperature                           | degrees Celsius * 10            |
| Dew point temperature                     | degrees Celsius * 10            |
| Sea level pressure                        | hectopascals                    |
| Wind direction                            | angular degrees                 |
| Wind speed                                | meters per second * 10          |
| Total cloud cover                         | coded, see format documentation |
| One-hour accumulated liquid precipitation | millimeters                     |
| Six-hour accumulated liquid precipitation | millimeters                     |

Further information on the data source can be found in the Documents directory of this repository. 

Weather data is aggregated to the county level by merging the ISD weather data with ![the 2019 US County lines published by the U.S. Census](https://www2.census.gov/geo/tiger/TIGER2019/COUNTY/). The merging is accomplished through conducting a within join on the coordinates of the weather pattern 
measurement stations within the ISD data source. 

The wildfire data is published by the ![National Interagency Fire Center](https://data-nifc.opendata.arcgis.com/datasets/nifc::interagency-fire-perimeter-history-all-years/about) and includes measurements on wildfire occurrence and acreage impacted. Wildfire boundaries were intersected with 2019 Us County lines in order to develop a metric for acreage of specific counties impacted  by wildfires on an annual basis. 

The final merged dataset includes minimum and maximum weather variables, number of wildfires, and total acreage damaged by wildfires by month and by county within the state of California between 2016 and 2020. 

## Exploratory Data Analysis 
In order to assess the extent to which patterns in specific weather variables may be divergent from normal patterns across all U.S. states (i.e. warmer temperatures in the Summer months and colder temperatures in the winter months), time series on the minimum and maximums across weather variables per day were compared for both California and Georgia.  

### Air Temperature 
![](https://github.com/christianmconroy/NOAA_Weather_Analysis/images/min_max_airremp_ca.png)
![](https://github.com/christianmconroy/NOAA_Weather_Analysis/images/min_max_airremp_ga.png)

- While both Georgia and California follow similar patterns of warm and cold temperatures - low temperatures around January, for example - there occurred to be more days with maximum temperatures below 0 in Georgia compared to California. This may be due to the fact that California's temperatures likely vary more widely than those of Georgia due to the larger area covered by the state. 
- The lowest temperature in California over the study period occurred in December 2017 while the lowest temperature in Georgia occurred in January 2018. These low temperatures occurred during the ![December 2017-January 2018 North American cold wave](https://en.wikipedia.org/wiki/December_2017%E2%80%93January_2018_North_American_cold_wave) when a polar vortex froze a large part of the United States. 
- The highest temperature in California over the study period occurred in late August 2020 while the highest temperature in Georgia occurred in Fall 2019. August 2020 set a ![record](https://www.latimes.com/california/story/2020-09-10/a-sizzling-record-august-was-hottest-month-on-record-in-california) for high temperatures in California.

### Wind Speeds 
![](https://github.com/christianmconroy/NOAA_Weather_Analysis/images/min_max_windsp_ca.png)
![](https://github.com/christianmconroy/NOAA_Weather_Analysis/images/min_max_windsp_ga.png)

- The highest wind speed for California over the reporting period occurred in early Summer 2017 and was higher than that of Georgia, which occurred in early Spring 2020. High winds have traditionally been a ![major contributing factor](https://www.washingtonpost.com/weather/2019/10/28/whats-driving-historic-california-high-wind-events-worsening-wildfires/) to the spread of wildfires in California. 
- Georgia appeared to have far more days with a minimum wind speed qualifying as calm or no wind than did California. 
- Cyclical trends in wind speeds were far less pronounced for Georgia than California. Wind speeds appeared to increase in particular during Spring and early Summer in California. 

### Sea Level Pressure
![](https://github.com/christianmconroy/NOAA_Weather_Analysis/images/min_max_sealevelpressure_ca.png)
![](https://github.com/christianmconroy/NOAA_Weather_Analysis/images/min_max_sealevelpressure_ga.png)

- The trajectories for minimum and maximum sea level pressure as measured by hectopascals were more similar for Georgia and California as compared to other analyzed weather variables. In both states, sea level pressure tended to rise in the Winter and fall during the Summer. 
- The two states even experienced the same significant trough in sea level pressure in early Winter 2017, with Georgia at only 990 hectopascals and California only slightly above that number. 
- Cyclical trends in wind speeds were far less pronounced for Georgia than California. Wind speeds appeared to increase in particular during Spring and early Summer in California. 

### Accumulated Liquid Precipitation
![](https://github.com/christianmconroy/NOAA_Weather_Analysis/images/min_max_preciphourly_ca.png)
![](https://github.com/christianmconroy/NOAA_Weather_Analysis/images/min_maxpreciphourly_ga.png)

- Georgia experienced many more days of higher rainfall as measured by liquid precipitation depth over one hour duration, with many days over the study period experiencing maximums of over 10 millimeters. In contrast, California did not experience a single day of liquid precipitation depth of over 10 millimeters during the study period. This makes sense given the dry conditions in California. 
- The highest liquid precipitation depth for California occurred in early Spring 2017, with the state receiving a maximum of just over 7 millimeters. This period of rainfall in California is well documented, and was at the tail end of the ![wettest Winter in California on record](https://www.kqed.org/news/11407012/the-rainy-season-of-2016-17-is-officially-one-for-the-record-books). 
- The highest liquid precipitation depth for Georgia occurred in early Summer 2017, when a liquid precipitation depth of over 30 millimeters was recorded in the state. Hurricane season was ![extremely active](https://patch.com/georgia/atlanta/extremely-active-2017-hurricane-season-updated-georgia-forecast) in Georgia in 2017, leading to increased precipitation that year. 
- Precipitation levels followed an expected trajectory in California over the study period, with maximum precipitation being comparatively low during the Summer months every year.

## Developing a Predictive Model for Wildfire Spread<br><sup> Methods and Evaluation</sup>

### The Target Variable
- Target created by intersecting wildfire boundaries with county lines
- The total wildfire acreage for the intersected county is calculated by multiplying the percent overlap of the wildfire boundary and the county and the total acreage of the wildfire. 
- The target variable is then summed up at the county-month level to provide the total amount of acreage burned in a county in a month due to wildfires. 
- The burned acreage is not distinct and therefore the total acreage burned can consist of the same area being burned multiple times if multiple wildfires occurred on that area in a month. 
- The final dataset consisted of 715 wildfires that took place in California between 2016 and 2020. 


### The Features

- 25 features serve as input variables for the model
- Feature variables consist of the maximum and minimum for each weather variable and one hot encoding for the month
- The mimumum for 360 wind direction and sky condition are both dropped from the model as they are consistently 0 for all observations.

### The Approach

- 5 different supervised learning methods are evaluated
  - K Nearest Neighbor
  - Linear Regression
  - Ridge Regression
  - Lasso Regression
  - Decision Tree
  - Random Forest
  - SGD Regression
  - Gradient Boosted Regression Tree
- Due to class imbalance (counties experiencing wildfires on a given month represented only 26.4% of the tenants overall), a downsampled versions of the data is evaluated as well.  
- Models are primarily compared based on the Root Mean Squared Error (RMSE) metric, which provides a measure of the normalized distance between the vector of predicted values and the vector of observed values.

### Model Results - Full Dataset

Overall, the RMSE for all models is consistently low. 

| Model*                           | AUC   | Precision | Recall    | F1  | Accuracy**  |
| -------------------------------- |:-----:| --------- | --------- | ----|-------------|      
| Logistic Regression              | 59.8% | 68%       | 21%       | 32% | 90.5%       |
| Decision Tree                    | 59.8% | 68%       | 21%       | 32% | 90.5%       |
| Random Forest                    | 60.5% | 40%       | 26%       | 31% | 87.9%       |
| Gradient Boosted Regression Tree | 58%   | 72%       | 17%       | 27% | 90.4%       |
| Support Vector Machine           | 60%   | 68%       | 21%       | 32% | 90.5%       | 

*\*The above results are based on evaluating the trained model on a hold-out test set*

*\*\*Accuracy is not a valuable metric here given imbalance with the target variable*


### Model Results - Upsampled Dataset

| Model*                           | AUC   | Precision | Recall | F1  | Accuracy**  |
| -------------------------------- |:-----:| --------- | -------| ----|-------------|      
| Logistic Regression              | 65.3% | 70%       | 54%    | 61% | 65.4%       |
| Decision Tree                    | 65.8% | 71%       | 53%    | 61% | 65.8%       |
| Random Forest                    | 94.1% | 90%       | 100%   | 94% | 94.1%       |
| Gradient Boosted Regression Tree | 66.2% | 71%       | 55%    | 62% | 66.2%       |
| Support Vector Machine           | 66.2% | 69%       | 56%    | 62% | 66.6%       | 

*\*The above results are based on evaluating the trained model on a hold-out test set*

*\*\*Accuracy is not a valuable metric here given imbalance with the target variable*


### Model Results - Downsampled Dataset

| Model*                           | AUC   | Precision | Recall | F1  | Accuracy**  |
| -------------------------------- |:-----:| --------- | -------| ----|-------------|      
| Logistic Regression              | 67.4% | 74%       | 56%    | 64% | 67.2%       |
| Decision Tree                    | 65.6% | 72%       | 52%    | 61% | 65.4%       |
| Random Forest                    | 61.6% | 62%       | 62%    | 62% | 61.6%       |
| Gradient Boosted Regression Tree | 67.6% | 73%       | 57%    | 64% | 67.4%       |
| Support Vector Machine           | 67%   | 75%       | 52%    | 62% | 66.6%       | 

*\*The above results are based on evaluating the trained model on a hold-out test set*

*\*\*Accuracy is not a valuable metric here given imbalance with the target variable*

## Conclusions
- Random Forest on the upsampled version of the data offered the best performance as measured by AUC (94.1%)
- Length of contract history, age at the start of contract history, and the use of cash for payment appeared to be most important in predicting bad tenants
- Evaluation on an upsampled version of the data showed significant performance gain

![](https://github.com/christianmconroy/Projects/blob/master/bad_tenant_ml_modeling/images/feature%20imp.png)
