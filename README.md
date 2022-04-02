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

1.	Air temperature (degrees Celsius * 10)
2.	Dew point temperature (degrees Celsius * 10)
3.	Sea level pressure (hectopascals)
4.	Wind direction (angular degrees)
5.	Wind speed (meters per second * 10)
6.	Total cloud cover (coded, see format documentation)
7.	One-hour accumulated liquid precipitation (millimeters)
8.	Six-hour accumulated liquid precipitation (millimeters)

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

### Dew point Temperature 
![](https://github.com/christianmconroy/NOAA_Weather_Analysis/images/min_max_dewtempt_ca.png)
![](https://github.com/christianmconroy/NOAA_Weather_Analysis/images/min_max_dewtempt_ga.png)

- The majority of tenants (6,622) have a contract history of at least 24 months
- The second largest number of tenants had a history of 12 months or shorter
- While 3 month contracts were permitted, few tenants selected that option
- Not all contract history consists of finished contracts as there is no contract end date listed for tenants still within their contract as of 2017-12-31
- Length of contract history is therefore defined as length from start of first contract to 2017-12-31 for any tenant still within a contract

### Payments and Transactions
![](https://github.com/christianmconroy/Projects/blob/master/bad_tenant_ml_modeling/images/num_pay_meth_ten.png)

- 9,839 tenants used only one payment method while 309 used 2
- Bank transfer was the most used payment method followed by direct debit and then cash
- The deposit is always 3 times the amount of the monthly rent
- For the contracts that have ended, 86.7% received the deposit back while 5.1% received a partial deposit back
- 8.2% forfeited their deposit

| Deposit Status        | Number of Contracts|
| --------------------- |:------------------:|
| Contract Not Ended Yet| 9,270              |
| Deposit Returned      | 1,988              |
| Partially Returned    | 117                |
| Not Returned          | 188                |

### Evictions and Bad Tenants

- 32 tenants appeared likely to have been evicted based on having gaps on:
  -The amount of rent they paid versus how much they owed based on contract start and end date or 12/31/2017
  - The number of payments short they were based on how many payments they should have made based on contract start and end date of 12/31/2017
- Almost all had a pattern of late rent payment and three-quarters paid the highest late fee on at least one occasion

| Likely Eviction Cause | Number of Tenants                                              |
| -------------------------------------------------------------------------------- |:---:|
| Late payment more than 3 months                                                  | 31  |
| Rent lateness hitting or exceeding largest late fee (Half of monthly rent extra) | 24  |

## Developing a Predictive Model for Bad Tenants<br><sup> Methods and Evaluation</sup>

### The Target Variable
- Target created by setting a 25% threshold on the average of 8 normalized features indicative of bad tenant behavior 
- 25% threshold used to capture “bad” tenants and not just tenants likely to get evicted
- Aggregated values based on whatever the time period is for the tenant 
- The labelling window encompasses the about 72-month period between the first contract date and the last transaction payments recorded on 12/31/17
- Because transactions begin 12/02/2014, it is assumed that earlier payments were paid on time and tenants were charged 3x deposit rate
- The final dataset has 1,203 bad tenants and 10,266 good tenants

| Bad Tenant Indicator                                                                              | Format                                                                                                              |
| ------------------------------------------------------------------------------------------------- |:------------------------------------------------------------------------------------------------------------------:|
| Rent amount short  based on expected amount between contract start and end or 12/31/17            | Amount                                                                                                              |
| Number of rent payments short based on expected amount between contract start and end or 12/31/17 | Volume                                                                                                              |
| Number of late payments                                                                           | Volume                                                                                                              |
| Average late fee percentage                                                                       | Percent                                                                                                            |
| Maximum late fee percentage                                                                       | Percent                                                                                                            |
| Latest payment                                                                                    | Days                                                                                                                |
| Deposit Status                                                                                    | 0 for deposit returned or contract term not yet ended; 0.5 for partial return, and 1 for non-return               |

### The Features

- 8 features serve as input variables for the model
- The aggregated values are based on whatever the time period is for the tenant 
- A control variable is included to indicate whether a tenant has concluded contract history or is still within contract as of 12/31/17
- Percentages are used to control for the length of contract history and payment methods for contracts that began before 12/02/2014
- The labelling window encompasses the 72-month period between the first contract date and the last transaction payments recorded on 12/31/17

| Feature                           | Format  |
| --------------------------------- |:-------:|
| Contract History                  | Days    |
| Age at Start of Contract History  | Years   |
| Number of Contracts               | Volume  |
| Number of Payment Methods         | Percent |
| Tenant Reached End of Contract    | Boolean |
| Direct Debit Portion of Payments  | Percent |
| Cash Portion of Payments          | Percent |
| Bank Transfer Portion of Payments | Percent |

### The Approach

- 5 different supervised learning methods are evaluated
  - Logistic Regression
  - Decision Tree
  - Random Forest
  - Gradient Boosted Regression Tree
  - Support Vector Machine
- Due to class imbalance (bad tenants represented only 10.5% of the tenants overall), both upsampled and downsampled versions of the data are evaluated
- L1 regularization, or lasso, is applied where relevant for feature selection 
- Feature importance plots are employed to provide explainability for models 
- Random Search and Random Grid are used for hyperparameter optimization 
- Models are primarily compared based on the Area Under the ROC Curve (AUC) metric providing an aggregate measure of performance across classification thresholds
- AUC is more useful than accuracy as an evaluation measure due to class imbalance 

### Model Results - Full Dataset

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
