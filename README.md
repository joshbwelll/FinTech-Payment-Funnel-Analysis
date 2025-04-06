# FinTech-Payment-Funnel-Analysis

**Executive Summary:**
A large volume of unpaid subscriptions at a fintech company revealed significant friction in the online payment experience, negatively impacting revenue. This analysis leveraged SQL and a data science notebook to build a comprehensive product funnel, identifying key drop-off points within the payment process. The findings informed strategic, data-driven recommendations to improve the conversion rate of successful payments.

![image](https://github.com/user-attachments/assets/b6b46878-c6fa-43d7-ab0d-cd783c0c7cdb)


**Business Problem:**
The finance team has noticed that many subscriptions haven't been paid for, so they've reached out to the product team to understand if there are any frictions points in the online payment portal so they can increase the conversion rate (% of subscriptions that are successfully converting to a paid subscription).

**Methodology:**
1. EDA
2. Product Funnel Analysis
3. Data Visualization (Tableau) 

**Skills:**
* SQL – CTEs, CASE, subqueries, window functions
* Data visualization
* Data Wrangling
* Data Cleaning
* Data Science Notebook
* Snowflake Data warehouse
* Tableau
* Data Model
* SQL Code:

```sql
WITH max_status_reached AS (
SELECT
    PSL.subscription_id,
    MAX(PSL.STATUS_ID) as max_status
FROM PUBLIC.PAYMENT_STATUS_LOG PSL
JOIN PUBLIC.PAYMENT_STATUS_DEFINITIONS DEF
ON PSL.STATUS_ID = DEF.STATUS_ID
GROUP BY 1   
)
,
payment_funnel AS (
SELECT
    subs.subscription_id,
    date_trunc('year', order_date) as order_year,
    current_payment_status,
    m.max_status,
    CASE WHEN m.max_status = 1 THEN 'Payment Widget Opened'
        WHEN m.max_status = 2 THEN 'Payment Entered'
        WHEN m.max_status = 3 and current_payment_status = 0 THEN 'User Error with Payment'
        WHEN m.max_status = 3 and current_payment_status != 0 THEN 'Payment Submitted'
        WHEN m.max_status = 4 and current_payment_status = 0 THEN 'Payment Processing Error with Vendor'
        WHEN m.max_status = 4 and current_payment_status != 0 THEN 'Payment Success w/Vendor'
        WHEN m.max_status = 5 THEN 'Complete'
        WHEN m.max_status is null THEN 'User Has Not Started the Payment Process'
        END AS payment_funnel_stage
FROM public.subscriptions subs
LEFT JOIN max_status_reached m
ON subs.subscription_id = m.subscription_id
)
SELECT
    payment_funnel_stage,
    order_year,
    COUNT(*) as numb_subs
FROM payment_funnel
GROUP BY 1, 2
ORDER BY 2 DESC
```


```sql
WITH max_status_reached AS (
SELECT
    PSL.subscription_id,
    MAX(PSL.STATUS_ID) as max_status
FROM PUBLIC.PAYMENT_STATUS_LOG PSL
JOIN PUBLIC.PAYMENT_STATUS_DEFINITIONS DEF
ON PSL.STATUS_ID = DEF.STATUS_ID
GROUP BY 1   
)
,
payment_funnel AS (
SELECT
    subs.subscription_id,
    date_trunc('year', order_date) as order_year,
    current_payment_status,
    m.max_status,

    CASE WHEN m.max_status = 5 THEN 1 ELSE 0 END AS completed_payment,
    CASE WHEN m.max_status is not null THEN 1 ELSE 0 END AS started_payment  
FROM public.subscriptions subs
LEFT JOIN max_status_reached m
ON subs.subscription_id = m.subscription_id
)
SELECT
    SUM(completed_payment) as num_subs_completed_payment,
    SUM(started_payment) as num_subs_started_payment,
    COUNT(*) as total_subs,
    num_subs_completed_payment * 100 / total_subs as conversion_rate,
    num_subs_completed_payment * 100 / num_subs_started_payment as workflow_completion_rate 

FROM payment_funnel
```


```sql
WITH error_subs AS (
    SELECT
        distinct subscription_id
    FROM public.payment_status_log
    WHERE status_id =0
)
SELECT
    COUNT(errs.subscription_id) * 100 / COUNT(s.subscription_id) AS perc_subs_hit_error
FROM public.subscriptions s
LEFT JOIN error_subs errs
ON s.subscription_id = errs.subscription_id
--
--
--can solve as subquery
--SELECT
--    (SELECT COUNT(distinct subscription_id) FROM public.payment_status_log WHERE status_id = 0) * 100 / COUNT(*) AS per_subs_hit
--FROM public.subscriptions
```


```sql
WITH error_subs AS (
    SELECT
        distinct subscription_id
    FROM public.payment_status_log
    WHERE status_id =0
)
SELECT
    s.subscription_id,
    CASE WHEN errs.subscription_id is not null THEN 1
    ELSE 0
    END AS has_error    
FROM public.subscriptions s
LEFT JOIN error_subs errs
ON s.subscription_id = errs.subscription_id
```


**Results & Business Recommendation:**

**Results:**
- [Tableau visualizations here]
- X% of subscriptions have hit an error
- X% of subscriptions have no opened the payment portal

**Business Recommendations:**
- Reduce friction on the enter payment page by considering Apple Pay, Google Pay, or other payment methods that don't require entering in a credit card every time. This will help reduce user errors due to incorrect payment info.
- Reach out to the 3rd party payment processing vendor and inquire about the errors on their side and determine a plan reduce those in the future.
Work with the product manager to increase the number of subscriptions that are opening the payment portal and attempting to pay. Since a large number of subscriptions aren't even going into the payment portal, we're losing a large number of opportunities at the beginning of the funnel, so maybe we can set up payment reminders or have customer service agent call them to encourage payment.
- Meet with Product Manager and UX/UI Design Team to consider making enhancements to the payment page to better highlight the next steps in the payment flow.

**Next Steps:**
* Investigate why subscriptions aren't even starting the payment process. 
* Conduct a deeper error type analysis to distinguish between user-side and vendor-side failures
* Investigate issues causing users to bypass the payment portal altogether. Is it a process issue on our side? Are customers forgetting? Potential technical barriers or lack of reminders?

