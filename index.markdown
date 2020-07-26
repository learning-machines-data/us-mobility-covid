---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
---

## **Background**
Covid-19 outbreak continues to spread rapidly and influence our daily work life. 
Due to multiple goverment laws and many other factors intended to slow spread of virus causing chnages in consumer foot traffic and its spend behavior.
This is causing demand in some category product increasing unexpectedly and at the same decreasing in others. 
We can get some insights on mobility from the data that Google presents in its COVID-19 Community Mobility Reports. 
Using anonymized data provided by apps such as Google Maps, the company has produced a regularly updated dataset that shows how peoples’ movements have changed throughout the pandemic.
This new dataset from Google measures visitor numbers to specific categories of location (e.g. grocery stores; parks; train stations) every day and compares this change relative to baseline day before the pandemic outbreak. 
Baseline days represent a normal value for that day of the week, given as median value over the five‑week period from January 3rd to February 6th 2020.
Measuring it relative to a normal value for that day of the week is helpful because people obviously often have different routines on weekends versus weekdays.
The reports chart movement trends over time by geography, across different categories of places such as retail and recreation, groceries and pharmacies, parks, transit stations, workplaces, and residential
More information can be found here:  

[Google Mobility Trends](https://ourworldindata.org/covid-mobility-trends)  
[US Community Mobility Reports](https://www.gstatic.com/covid19/mobility/2020-07-21_US_Mobility_Report_en.pdf)

## **Objective**
To predict spend behavior based on covid impact and consumer mobility on county level

## **Techniques used**
- Python
- Regression Models
- Tableau

## **Data Source**
- Information from a variety of openly available world government data sources and curated datasets
- County Level mobility data
- County Level spending data
- Unemployment insurance claims data from the Department of Labor (national and state-level) and numerous individual state agencies (county-level)
	
## **Process Flow**
- Import county level covid data

```python
covid_url = 'https://coronadatascraper.com/timeseries.csv'
df_covid = pd.read_csv(covid_url,low_memory=False)
df_covid.date = pd.to_datetime(df_covid.date, infer_datetime_format=True)
df_covid_us = df_covid[df_covid['country'] == 'United States']
df_covid_us = df_covid_us.dropna(subset=['county'])
df_covid_us = df_covid_us[df_covid_us['aggregate'] =='county']
df_covid_us  = df_covid_us.reset_index(drop=True)
null_cols = df_covid_us.isna().sum()
remove_cols = [col for col in null_cols.index if null_cols[col]>15000 and col!='deaths']

df_covid_us = df_covid_us.rename(columns={'county':'COUNTY', 'state':'STATE'})
df_covid_us = df_covid_us.drop(columns = ['name', 'level', 'city', 
                                          'country','url','tz','population','aggregate']+remove_cols)
df_covid_us = df_covid_us.fillna(0)
df_covid_growth = df_covid_us.groupby(['COUNTY', 'STATE'])['growthFactor'].mean().reset_index()
```

- Import county level mobility data

```python
mob_index_url = 'https://raw.githubusercontent.com/descarteslabs/DL-COVID-19/master/DL-us-mobility-daterow.csv'
df_mobility_index = pd.read_csv(mob_index_url)
df_mobility_index = df_mobility_index.dropna(subset=['fips'])
df_mobility_index.fips = df_mobility_index.fips.astype(int)
df_mobility_index.date = pd.to_datetime(df_mobility_index.date,infer_datetime_format=True)
df_mobility_index.fips = df_mobility_index.fips.astype(str).str.zfill(5)

df_mobility_index = df_mobility_index.dropna(subset=['admin2'])
df_mobility_index = df_mobility_index.rename(columns={'admin1':'STATE','admin2':'COUNTY' })
df_mobility_index = df_mobility_index.drop(columns = ['country_code', 'admin_level'])
```

- Daily moility trend 
![Mobility Trend](images/us-mobility-trend.png)

- Import county level spend data from Affinity Solutions
```python
county_spending_url =  'https://raw.githubusercontent.com/Opportunitylab/EconomicTracker/main/data/Affinity%20-%20County%20-%20Daily.csv'
df_county_spending = pd.read_csv(county_spending_url)

df_county_spending['date'] = df_county_spending['year'].astype(str) + '-' + \
                            df_county_spending['month'].astype(str) + '-' + \
                            df_county_spending['day'].astype(str)
df_county_spending.date = pd.to_datetime(df_county_spending.date, infer_datetime_format=True)
df_county_spending.countyfips = df_county_spending.countyfips.astype(int)
df_county_spending.countyfips = df_county_spending.countyfips.astype(str).str.zfill(5)
df_county_spending = df_county_spending.rename(columns={'countyfips':'fips'})
```

- Import unemployment insurance claims data from the Department of Labor (national and state-level) and numerous individual state agencies (county-level).
```python

ui_url = 'https://raw.githubusercontent.com/Opportunitylab/EconomicTracker/main/data/UI%20Claims%20-%20County%20-%20Weekly.csv'
df_ui = pd.read_csv(ui_url)
df_ui['date'] = df_ui['year'].astype(str) + '-' + \
                df_ui['month'].astype(str) + '-' + \
                df_ui['day_endofweek'].astype(str)
df_ui.date = pd.to_datetime(df_ui.date, infer_datetime_format=True)
df_ui.initial_claims_rate = '0' + df_ui.initial_claims_rate.astype(str)
df_ui.initial_claims_rate = 3 - df_ui.initial_claims_rate.astype(float)
df_ui.countyfips = df_ui.countyfips.astype(str).str.zfill(5)
```

- Create 7-day rolling of mobility
```python
mob_col= ['m50']
m50 = df_mobility_index.groupby(['fips','date'])[mob_col].mean()
m50['rolling_mean_mob'] = m50[mob_col].rolling(7,min_periods=1).mean()
m50 = m50.reset_index()
```

![County Level Spend and Mobility](images/county-level-spend-and-mobility.png)

- Linear Correlation between Consumer Spending and Mobility on County Level
![Correlation between Spend and Mobility](images/linear-correlation-spend-mobility.png)

- Merge county spending data and mobility data and understand the correlation between spend and mobility at county level
```python
df_merged = pd.merge(left = m50, right = df_county_spending, on = ['fips', 'date'], how = 'outer')
def get_county_state(val_dict,x):
    return val_dict[x]

df_county_geocodes = pd.read_csv('./Data/County_GEOCODES-v2017.csv',encoding='latin')
df_county_geocodes['fips'] = df_county_geocodes['fips'].astype(str).str.zfill(5)
county_dict = df_county_geocodes.set_index('fips')['long_name'].to_dict()
# Pearson correlation between consumer spending and mobility
df_mobility_spend_corr = df_merged.groupby(['fips'])[['m50','spend_all']].corr().reset_index()
df_mobility_spend_corr = df_mobility_spend_corr[df_mobility_spend_corr['level_1']=='m50']
df_mobility_spend_corr = df_mobility_spend_corr.drop(columns=['level_1', 'm50'])
df_mobility_spend_corr = df_mobility_spend_corr.rename(columns={'spend_all':'Mob-Spend Corr'})

df_mobility_spend_corr['loc'] = df_mobility_spend_corr['fips'].apply(lambda x: get_county_state(county_dict,x))
df_mobility_spend_corr['Mob-Spend Corr'] = df_mobility_spend_corr['Mob-Spend Corr'].round(2)
```
- Check reduction in spending and mobility before and after covid
```python
target_cols = ['rolling_mean_mob','spend_all']
covid_date = datetime.datetime.strptime('2020-03-15', '%Y-%m-%d')
window = covid_date -datetime.timedelta(days=10)
df_covid = df_merged[df_merged.date>=window]
df_pre = df_merged[(df_merged.date<window)]
df_pre_mean = df_pre.groupby(['fips','date'])[target_cols].mean()

pre_covid_mean = df_pre_mean.groupby(level=[0])[target_cols].mean()
df_covid_mean = df_covid.groupby(['fips','date'])[target_cols].mean()


post_covid_min_date = df_covid_mean.groupby(level=[0])[target_cols].idxmin()
for col in target_cols:
    rename_col = 'min_date_' + col
    post_covid_min_date['min_date_'+col] = post_covid_min_date[col].str[1]
    post_covid_min_date = post_covid_min_date.drop(columns=[col])
post_covid_min = y.groupby(level=[0])[target_cols].min()
for col in target_cols:
    post_covid_min = post_covid_min.rename(columns={col:'Min_'+col})
df_mob_spend_red = pd.concat([pre_covid_mean,post_covid_min,post_covid_min_date],axis=1)

df_mob_spend_red = df_mob_spend_red.reset_index()
df_mob_spend_red = df_mob_spend_red.rename(columns={'index':'fips'})
for col in target_cols:
    min_date_col = 'min_date_' + col
    
    df_mob_spend_red['Drop_days_'+col] = (df_mob_spend_red[min_date_col] - covid_date).dt.days
    if col!='spend_all':
        df_mob_spend_red['Norm_Drop_Rate_'+col] = ((df_mob_spend_red[col] -df_mob_spend_red['Min_'+col])/df_mob_spend_red['Drop_days_'+col])/df_mob_spend_red[col]
        df_mob_spend_red['Pct_Red_'+col] =( df_mob_spend_red[col] - df_mob_spend_red['Min_'+col])/df_mob_spend_red[col]
    else:
        df_mob_spend_red['Norm_Drop_Rate_'+col] = ((-1*df_mob_spend_red['Min_'+col])/df_mob_spend_red['Drop_days_'+col])
        df_mob_spend_red['Pct_Red_'+col] = -1*df_mob_spend_red['Min_'+col]
df_mob_spend_red = df_mob_spend_red[df_mob_spend_red['Pct_Red_rolling_mean_mob']>=-0.5]
```

![Reduction in Mobility and Spend](images/reduction-in-spend-and-mobility.png)

## **Exploratory Data Analysis**

## **Conclusion**
