# 📊 Insurance Data Warehouse Analytics — Portfolio Project

## 📂 Database Setup

USE datawarehouseanalytics;

SET SESSION sql_mode = (
    SELECT REPLACE(@@sql_mode, "ONLY_FULL_GROUP_BY", "")
);

Explanation:
	1.	USE datawarehouseanalytics → Switches the active database.
	2.	SET SESSION sql_mode → Temporarily removes ONLY_FULL_GROUP_BY for the session so we can run flexible group queries.

⸻

📄 Data Overview

Tables

policyholders
	•	policyholder_id — Unique ID.
	•	full_name — Name.
	•	gender, date_of_birth, address, phone_number, email.

policies
	•	policy_id — Unique ID.
	•	policyholder_id — FK to policyholders.
	•	policy_type, start_date, end_date, premium_amount, coverage_amount, status.

claims
	•	claim_id — Unique ID.
	•	policy_id — FK to policies.
	•	claim_date, claim_type, claim_amount, approved_amount, claim_status.

⸻

# 1️⃣ Highest Average Claim Amount by Policy Type

SELECT policy_type,
       ROUND(AVG(claim_amount), 2) AS avg_claimamt
FROM claims
JOIN policies USING (policy_id)
GROUP BY 1
ORDER BY 2 DESC;

Result:

Policy Type	Avg Claim Amount
Group	5459.74
Individual	5221.90
Family	5166.55

Insight: Group policies have the highest average claims — possible higher risks or more expensive claims.

⸻

# 2️⃣ Rejected Claims in the Last 12 Months

SELECT 
    claim_type,
    SUM(CASE WHEN claim_status = 'Rejected' THEN 1 ELSE 0 END) AS cnt
FROM claims
WHERE YEAR(claim_date) > DATE_SUB(CURDATE(), INTERVAL 1 YEAR)
GROUP BY 1
ORDER BY 2 DESC;

Result:

Claim Type	Rejections
Consultation	20
Medication	20
Surgery	19
Hospitalization	17

Insight: Consultation & Medication rejections suggest stricter review processes for these categories.

⸻

# 3️⃣ Top 10 Policyholders by Approved Claims

SELECT 
    policyholder_id,
    full_name,
    ROUND(SUM(claim_amount), 2)
FROM claims
JOIN policies USING (policy_id)
LEFT JOIN policyholders USING (policyholder_id)
WHERE claim_status = 'Approved'
GROUP BY 2
ORDER BY 3 DESC
LIMIT 10;

Top 3 Results:
	1.	Robert Moore — 51,827.92
	2.	Jeffery Barber — 36,468.77
	3.	Jose Obrien — 34,196.12

⸻

# 4️⃣ Distribution of Policyholders by Age Group & Gender

SELECT 
    gender,
    CASE 
        WHEN TIMESTAMPDIFF(YEAR, date_of_birth, CURDATE()) BETWEEN 18 AND 25 THEN '18-25'
        WHEN TIMESTAMPDIFF(YEAR, date_of_birth, CURDATE()) BETWEEN 26 AND 35 THEN '26-35'
        WHEN TIMESTAMPDIFF(YEAR, date_of_birth, CURDATE()) BETWEEN 36 AND 45 THEN '36-45'
        WHEN TIMESTAMPDIFF(YEAR, date_of_birth, CURDATE()) BETWEEN 46 AND 60 THEN '46-60'
        ELSE '60+'
    END AS age_group,
    COUNT(*) 
FROM policyholders
GROUP BY gender, age_group
ORDER BY age_group;

Example Result (Text Bar Chart):

18-25  Female ████████████   12  
        Male   ██████████████ 14  
26-35  Female ███████████████████ 20  
        Male   ███████████████     16  
...


⸻

# 5️⃣ Active Policies with No Claims in the Last Year

SELECT COUNT(*) 
FROM (SELECT * FROM policies WHERE status = 'Active') po 
LEFT JOIN policyholders ph USING (policyholder_id) 
WHERE NOT EXISTS (
    SELECT 1 
    FROM claims c 
    WHERE policy_id = po.policy_id 
    AND claim_date >= DATE_SUB(CURDATE(), INTERVAL 1 YEAR)
);

Result: 76 policies.

⸻

# 6️⃣ Claim Approval Rate (%) per Claim Type

SELECT 
    claim_type,
    COUNT(*) / (SELECT COUNT(*) FROM claims c WHERE c.claim_type = cs.claim_type) * 100 AS approval_rate
FROM claims cs
WHERE claim_status = 'Approved'
GROUP BY claim_type;

Result:

Claim Type	Approval %
Consultation	37.11
Hospitalization	35.43
Surgery	30.72
Medication	34.78


⸻

# 7️⃣ Policy Status Analysis

SELECT 
    policy_type,
    ROUND(100 * SUM(CASE WHEN status = 'Expired' THEN 1 ELSE 0 END) / COUNT(*), 2) AS Cancelled_policies,
    ROUND(100 * SUM(CASE WHEN status = 'Cancelled' THEN 1 ELSE 0 END) / COUNT(*), 2) AS Lapsed_policies,
    AVG(premium_amount) AS avg_premium
FROM policies
GROUP BY 1;


⸻

# 8️⃣ Average Days from Policy Start to First Claim

WITH cte AS (
    SELECT policy_id, MIN(claim_date) AS first_claim
    FROM claims
    GROUP BY policy_id
)
SELECT AVG(TIMESTAMPDIFF(DAY, start_date, first_claim))
FROM policies
JOIN cte USING (policy_id);

Result: 141.57 days (~4.65 months).

⸻

# 9️⃣ Policyholders with Multiple Policies

SELECT 
    full_name,
    COUNT(DISTINCT policy_id),
    ROUND(SUM(premium_amount), 2) AS total_premium,
    ROUND(SUM(claim_amount), 2) AS total_claim
FROM policyholders
JOIN policies USING (policyholder_id)
LEFT JOIN claims USING (policy_id)
GROUP BY 1
HAVING COUNT(DISTINCT policy_id) > 1
ORDER BY 1;

⸻

# 🔟 Average Premium and Claim Payout Ratio per Policy Type

SELECT 
    policy_type,
    ROUND(avg_prem / avg_claim, 2) AS ratio
FROM 
    (SELECT 
        policy_type,
        AVG(premium_amount) AS avg_prem,
        AVG(claim_amount) AS avg_claim
     FROM 
        policies 
        JOIN claims USING (policy_id)
     GROUP BY 
        policy_type
    ) a;

⸻

## Insight: Identifies loyal or high-value customers who may deserve retention benefits.

⸻
## Key Takeaways
	•	Group policies show the highest average claim amounts.
	•	Consultation & Medication claims face higher rejection rates.
	•	A significant number (76) of active policies had no claims in the past year.
	•	High-value claim customers can be targeted for risk management or loyalty rewards.










