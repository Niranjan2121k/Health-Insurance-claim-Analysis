# 📊 Insurance Data Analytics — SQL Portfolio

> This project contains 10 analytic SQL queries run (or intended to run) against the `datawarehouseanalytics` database.  
> Where results were provided during the session they are included. Where I could not run the query (no DB access) I note that and give instructions to get the values.

---

## Setup (session-level)
```sql
USE datawarehouseanalytics;
SET SESSION sql_mode = (
    SELECT REPLACE(@@sql_mode, "ONLY_FULL_GROUP_BY", "")
);

Why: switch to the dataset and remove ONLY_FULL_GROUP_BY for the session so older/looser GROUP BY queries run without strict mode errors.

Data model (tables used)
	•	policyholders(policyholder_id, full_name, gender, date_of_birth, address, phone_number, email)
	•	policies(policy_id, policyholder_id, policy_type, start_date, end_date, premium_amount, coverage_amount, status)
	•	claims(claim_id, policy_id, claim_date, claim_type, claim_amount, approved_amount, claim_status)

Queries, explanations & results

---

# 1 Highest Average Claim Amount by Policy Type

SQL

SELECT policy_type,
       ROUND(AVG(claim_amount), 2) AS avg_claimamt
FROM claims
JOIN policies USING (policy_id)
GROUP BY 1
ORDER BY 2 DESC;

What it does: average claim per policy type — useful to know which product bears higher average payout.

Result (provided):

policy_type	avg_claimamt
Group	5459.74
Individual	5221.90
Family	5166.55

Text bar (scaled):

Group      ████████████████████████  5459.74
Individual ██████████████████████    5221.90
Family     █████████████████████     5166.55

2️⃣ Rejected Claims in the Last 12 Months — Common Types

SQL

SELECT 
    claim_type,
    SUM(CASE WHEN claim_status = 'Rejected' THEN 1 ELSE 0 END) AS cnt
FROM claims
WHERE claim_date >= DATE_SUB(CURDATE(), INTERVAL 1 YEAR)
GROUP BY 1
ORDER BY 2 DESC;

(Note: I changed WHERE YEAR(claim_date) > DATE_SUB(...) to a direct date comparison to include the last 12 months correctly.)

What it does: counts rejected claims by claim type in last 12 months.

Result (provided):

claim_type	cnt
Consultation	20
Medication	20
Surgery	19
Hospitalization	17

Text bar:

Consultation    ████████████████████████  20
Medication      ████████████████████████  20
Surgery         ███████████████████████   19
Hospitalization ████████████████████      17

Insight: Consultation & Medication have the highest rejection counts — review documentation rules or common rejection reasons.

3️⃣ Top 10 Policyholders with Highest Total Approved Claims

SQL

SELECT 
    policyholder_id,
    full_name,
    ROUND(SUM(claim_amount), 2) AS total_approved_claims
FROM claims
JOIN policies USING (policy_id)
LEFT JOIN policyholders USING (policyholder_id)
WHERE claim_status = 'Approved'
GROUP BY full_name, policyholder_id
ORDER BY total_approved_claims DESC
LIMIT 10;

What it does: finds highest-cost customers (approved claims total).

Top 10 result (provided):

Robert Moore      ████████████████████████  51827.92
Jeffery Barber    ████████████████░░░░░░░░  36468.77
Jose Obrien       ███████████████░░░░░░░░░  34196.12
Erica Patton      ██████████░░░░░░░░░░░░░░  25889.75
Brittany Park     █████████░░░░░░░░░░░░░░░  23886.12
Brian Ballard     █████████░░░░░░░░░░░░░░░  23617.81
Gregory Castillo  █████████░░░░░░░░░░░░░░░  23236.58
Jessica Davis     ████████░░░░░░░░░░░░░░░░  22561.02
Ryan Aguirre      ███████░░░░░░░░░░░░░░░░░  20746.52
Madison Nunez     ██████░░░░░░░░░░░░░░░░░░  19872.73

Insight: These policyholders are high cost — worth reviewing for care management, fraud detection, or special retention strategies.

4️⃣ Distribution of Policyholders by Age Group & Gender

SQL

SELECT 
    gender,
    CASE 
        WHEN TIMESTAMPDIFF(YEAR, date_of_birth, CURDATE()) BETWEEN 18 AND 25 THEN '18-25'
        WHEN TIMESTAMPDIFF(YEAR, date_of_birth, CURDATE()) BETWEEN 26 AND 35 THEN '26-35'
        WHEN TIMESTAMPDIFF(YEAR, date_of_birth, CURDATE()) BETWEEN 36 AND 45 THEN '36-45'
        WHEN TIMESTAMPDIFF(YEAR, date_of_birth, CURDATE()) BETWEEN 46 AND 60 THEN '46-60'
        ELSE '60+'
    END AS age_group,
    COUNT(*) AS cnt
FROM policyholders
GROUP BY gender, age_group
ORDER BY age_group;

Result (provided):

Gender	Age Group	Count
Female	18-25	12
Male	18-25	14
Female	26-35	20
Male	26-35	16
Female	36-45	15
Male	36-45	20
Female	46-60	16
Male	46-60	26
Female	60+	33
Male	60+	28

Text (clustered) bar chart (scaled to max = 33):

18-25  Female ████████████   12
       Male   ██████████████ 14

26-35  Female ███████████████████ 20
       Male   ███████████████     16

36-45  Female █████████████   15
       Male   ███████████████████ 20

46-60  Female ███████████████ 16
       Male   ███████████████████████ 26

60+    Female █████████████████████████████ 33
       Male   ███████████████████████        28

Insight: Strong representation in older age groups (60+), with slightly more females 60+.

5️⃣ Count of Active Policies with No Claims in the Last Year

SQL

SELECT COUNT(*)
FROM (SELECT * FROM policies WHERE status = 'Active') po
LEFT JOIN policyholders ph USING (policyholder_id)
WHERE NOT EXISTS (
    SELECT 1
    FROM claims c
    WHERE c.policy_id = po.policy_id
      AND c.claim_date >= DATE_SUB(CURDATE(), INTERVAL 1 YEAR)
);

Result (provided):

count = 76

Interpretation: 76 active policies had no claims in the past 12 months — potential indicator of low utilization or healthy customers.

6️⃣ Claim Approval Rate (%) per Claim Type

SQL

SELECT 
    claim_type,
    COUNT(*) / (SELECT COUNT(*) FROM claims c WHERE c.claim_type = cs.claim_type) * 100 AS approval_rate
FROM claims cs
WHERE claim_status = 'Approved'
GROUP BY claim_type;

What it does: calculates approved / total for each claim type.

Result (provided & rounded):

claim_type	approval_rate (%)
Consultation	37.11
Hospitalization	35.43
Surgery	30.72
Medication	34.78

Text bar:

Consultation    ██████████████████████  37.11%
Hospitalization ████████████████████    35.43%
Surgery         ██████████████████      30.72%
Medication      ████████████████████    34.78%

Insight: Consultation has the highest approval rate in this dataset sample.

7️⃣ Policy Status Analysis — % Expired, % Cancelled, & Avg Premium

SQL

SELECT 
    policy_type,
    ROUND(100 * SUM(CASE WHEN status = 'Expired' THEN 1 ELSE 0 END) / COUNT(*), 2) AS expired_pct,
    ROUND(100 * SUM(CASE WHEN status = 'Cancelled' THEN 1 ELSE 0 END) / COUNT(*), 2) AS cancelled_pct,
    ROUND(AVG(premium_amount), 2) AS avg_premium
FROM policies
GROUP BY policy_type;

Result (provided):

policy_type	expired_pct	cancelled_pct	avg_premium
Family	36.11	27.08	2969.18
Group	32.28	35.43	2941.70
Individual	34.11	27.13	3024.83

Text group bars (approx):

Family
  Expired (%)   ████████████████████ 36.11
  Cancelled (%) ████████████████     27.08
Group
  Expired (%)   █████████████████    32.28
  Cancelled (%) ██████████████████   35.43
Individual
  Expired (%)   ██████████████████   34.11
  Cancelled (%) ████████████████     27.13

Insight: Group shows higher cancelled % (35.43) vs expired; average premium is similar across types (~2.9k–3.0k).

8️⃣ Average Days from Policy Start to First Claim

SQL

WITH cte AS (
    SELECT policy_id, MIN(claim_date) AS first_claim
    FROM claims
    GROUP BY policy_id
)
SELECT 
    ROUND(AVG(TIMESTAMPDIFF(DAY, start_date, first_claim)), 2) AS avg_days_to_first_claim
FROM policies
JOIN cte USING (policy_id);

Result (provided):

avg_days_to_first_claim = 141.57 days  (≈ 4.65 months)

Insight: On average, first claim occurs ~141.6 days after policy start.

9️⃣ Premium-to-Claim Payout Ratio per Policy Type

SQL

SELECT policy_type,
       ROUND(avg_prem / NULLIF(avg_claim, 0), 2) AS premium_to_claim_ratio
FROM (
    SELECT policy_type,
           AVG(premium_amount) AS avg_prem,
           AVG(claim_amount)   AS avg_claim
    FROM policies
    JOIN claims USING (policy_id)
    GROUP BY policy_type
) a;

What it does: For each policy type it computes avg(premium_amount)/avg(claim_amount) — values >1 indicate premiums on average exceed claim costs.

Result: NOT PROVIDED in this session.

I do not have your DB access here to run the query and produce numeric ratios.
How you can get it: run the SQL above in your MySQL client and paste the output here — I’ll format the table and add a bar chart.

🔟 Policyholders with Multiple Policies — total premium & claim amount

(Number emoji style)

SQL

SELECT 
    full_name,
    COUNT(DISTINCT policy_id)        AS num_policies,
    ROUND(SUM(premium_amount), 2)    AS total_premium,
    ROUND(SUM(claim_amount), 2)      AS total_claim
FROM policyholders
JOIN policies USING (policyholder_id)
LEFT JOIN claims USING (policy_id)
GROUP BY full_name
HAVING COUNT(DISTINCT policy_id) > 1
ORDER BY total_premium DESC;

What it does: finds policyholders who hold >1 policy and sums their premium collected and claims paid.

Wrap-up — Key takeaways
	•	Group policies have the highest average claim amount (5,459.74).
	•	Consultation and Medication generate the most rejections (20 each) in the last 12 months.
	•	Top costly customers (approved claims) are identified (Robert Moore highest at 51,827.92).
	•	Average time to first claim ≈ 141.57 days.
	•	Policy status blends (Expired vs Cancelled) differ by policy type; Group has higher cancelled %.
	•	Two queries (premium-to-claim ratio & the list of policyholders with multiple policies) need running on your DB — I can format results if you paste them here.



