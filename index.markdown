---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
---

## **Background**


## **Objective**

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

## **Exploratory Data Analysis**

## **Conclusion**
