# 20231204_Cohort_Analysis

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
No dupplicates were found. None of the users have subscribed to multiple plan or with multiple currency.


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
