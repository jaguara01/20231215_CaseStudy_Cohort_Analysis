# 20231215 Cohort Analysis Case


AB Gaming is a digital gaming platform that offers a wide variety of games available
under monthly subscriptions. There are 3 different plans, labeled SMALL, MEDIUM and
LARGE, and can be paid in USD or EUR.
You are provided with 2 datasets:
- sales.csv: contains the sales of clients acquired since 2019-01-01. Data was
extracted on 2020-12-31. This dataset includes a unique identifier for each user
(account_id).
- user_activity.csv: this dataset contains the following user characteristics as well
as their unique identifier (same as in sales.csv):
	- gender: gender of user reported in their profile.
	-  age: age in completed years of the user at the beginning of their subscription.
	- type: device type the user has installed the gaming platform.
	- genre1: most played game genre by the user.
	- genre2: second most played game genre by the user.
	- hours: mean number of hours played by the user weekly.
	- games: median number of different games played by the user weekly

## Part 1: 
Use sales.csv dataset to perform a monthly cohort analysis of subscribers.
In a cohort analysis the data set is broken down into related groups rather than looking
at all users as one unit. These related groups, or cohorts, usually share common
characteristics or experiences within a defined time-span (taken from Wikipedia).
In this case we want to group users by the month of their initial purchase.
- What products have higher retention?
- Is there any difference between subscribers using the 2 currencies?

Deliverables:
- SQL queries that allow us to make a monthly cohort analysis.
- Create a chart showing the cohort
  
Before going throught the analysis, we want to realize some basic checks.

### Check unicity in term of plans and currency
```SQL
with dist AS
(
SELECT distinct account_id,
		plan,
        currency
 from sales
  )
  SELECT account_id,
		plan,
        currency,
        count(*)
  from DIST
  group by account_id,
		plan,
        currency
 having count(*) > 1;
```
No dupplicates were found. None of the users have subscribed to multiple plans or with multiple currencies. They don't switch from an plan to another one.

### Check unicity in term of transaction per month per user
```SQL
with dist AS
(
SELECT distinct account_id,
		DATE_PART('YEAR', to_date(start_date,'YYYY-MM-DD')) AS YEAR,
  		DATE_PART('MONTH', to_date(start_date,'YYYY-MM-DD')) AS MONTH
 from sales
  )
  SELECT account_id,
		YEAR,
        MONTH,
        count(*)
  from DIST
  group by account_id,
  		YEAR,
		MONTH
 having count(*) > 1
 order by account_id, year, month;
 ```
No dupplicates were found, the users have a maximum of one subscription per month. They don't change in the month of their plan.

## Monthly cohort analysis 

The following query take as an input the SALES table and produce a monthly cohort analysis. 
By default, we have an output of the entire set of data. By uncommenting some part of the code, we can have a representation by type of plans or by Currency.
```SQL
 with cohort_items as (
  select
        DISTINCT
        account_id,
        to_date(SUBSTRING(MIN(start_date) OVER (PARTITION BY account_id),1,7),'YYYY-MM') AS cohort_yymm -- Evaluate the cohort month
  from sales
),
cohort_index AS
(
  select
    A.account_id,
    -- A.plan,
    -- A.currency,

    cohort_yymm,
    (DATE_PART('YEAR', to_date(start_date,'YYYY-MM-DD')) - DATE_PART('YEAR', cohort_yymm)) * 12+
    (DATE_PART('MONTH', to_date(start_date,'YYYY-MM-DD')) - DATE_PART('MONTH',cohort_yymm)) 
     	AS cohort_index -- Cohort index as the difference between the initial purchase month and the purchase month
  from sales A

  left join cohort_items C 
  	ON A.account_id = C.account_id
    where 1=1
  	-- and currency='USD'
  	-- and plan = 'LARGE'

  ),

cohort_retention AS
(
  select --plan,
  		--currency,
  		cohort_yymm,
  		cohort_index,
        count(*) AS cohort_ret_nbr
  		
  from cohort_index
  group by cohort_yymm,
  		cohort_index
  		-- ,currency
  		-- ,plan
  order by cohort_yymm,
  		cohort_index
 )
 select *,
 		CAST(cohort_ret_nbr AS FLOAT)*100/(MAX(cohort_ret_nbr) OVER (PARTITION BY cohort_yymm)) AS cohort_ret_pct
 from cohort_retention
        
        
```

The output is the following:

![image](https://github.com/jaguara01/20231204_Cohort_Analysis/assets/134049731/783ddaa1-1b48-4abc-a622-5504128a4cb5)

and the full output table can be found [here](Part_1_Cohort_Retention.csv)

## Visualization:
The visualiasation of the cohort is made with tableau. The dashboard can be found via this link: [Cohort_retention](https://public.tableau.com/app/profile/alexis/viz/Cohort_retention_17016768706120/Dashboard1)

![image](https://github.com/jaguara01/20231204_Cohort_Analysis/assets/134049731/6200f3a3-4182-4758-b42b-9036fb300420)

### Currency
Here is a visualization for each currency. We can see that the retention rate is often slightly better in USD than in EUR.

We also observe that information about returning clients are missing in the currency EUR for the last two months (2020-11 and 2020-12). This can be explain by a problem while collecting data or a delay in the data coming from the european market.

![image](https://github.com/jaguara01/20231204_Cohort_Analysis/assets/134049731/c160937b-b645-4944-865b-4b6ab4dae84c)

![image](https://github.com/jaguara01/20231204_Cohort_Analysis/assets/134049731/1e844678-edfb-41c6-aa2a-5bb4418c730f)

the following query confirm our observations:

```SQL

 with currency as
 (
 select CURRENCY,
 		SUBSTRING(MIN(start_date) OVER (PARTITION BY account_id),1,7) as cohort_dt,
        SUBSTRING(start_date,1,7) as purchase_dt
 from sales
   )
 SELECT cohort_dt,
 		purchase_dt,
        sum(case when currency = 'EUR' THEN 1 ELSE 0 END) AS EUR,
        sum(case when currency = 'USD' THEN 1 ELSE 0 END) AS USD
  FROM currency
  where purchase_dt in ('2020-11','2020-12')
  group by cohort_dt,
 		purchase_dt

```

![image](https://github.com/jaguara01/20231204_Cohort_Analysis/assets/134049731/562738b5-64f6-4402-a739-eb27650905e0)

### Subscription plans:
We observe that the SMALL plan is the one attracting the most people and the LARGE plan is the one with the least number of subscription.
We can also conclude the the SMALL plan is the one with the highest rentention rate over time.

![image](https://github.com/jaguara01/20231204_Cohort_Analysis/assets/134049731/a59e6e6b-9719-4afb-ad3c-08a8f97ed946)

![image](https://github.com/jaguara01/20231204_Cohort_Analysis/assets/134049731/3643a23c-cb7a-4c69-b207-c2a4d3cc9a50)

![image](https://github.com/jaguara01/20231204_Cohort_Analysis/assets/134049731/b7a21693-dcb5-4b96-b3b7-1dbc75f25bf8)








