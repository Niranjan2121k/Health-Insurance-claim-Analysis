# 📊 Insurance Data Analytics — SQL Portfolio

## Project Overview
This portfolio showcases 10 analytical SQL queries performed on an insurance database, demonstrating data exploration, aggregation, and business intelligence capabilities. The analysis covers policy performance, claim patterns, customer segmentation, and operational metrics.

## Database Schema
The analysis uses three main tables from the `datawarehouseanalytics` database:

- **policyholders** (`policyholder_id`, `full_name`, `gender`, `date_of_birth`, `address`, `phone_number`, `email`)
- **policies** (`policy_id`, `policyholder_id`, `policy_type`, `start_date`, `end_date`, `premium_amount`, `coverage_amount`, `status`)
- **claims** (`claim_id`, `policy_id`, `claim_date`, `claim_type`, `claim_amount`, `approved_amount`, `claim_status`)

## Technical Setup
```sql
USE datawarehouseanalytics;
SET SESSION sql_mode = (
    SELECT REPLACE(@@sql_mode, "ONLY_FULL_GROUP_BY", "")
);
```
*Removed ONLY_FULL_GROUP_BY restriction to accommodate flexible GROUP BY queries*

## Analytical Queries & Results

### 1. Highest Average Claim Amount by Policy Type
**Objective**: Identify which policy type generates the highest average claim payout.

```sql
SELECT policy_type,
       ROUND(AVG(claim_amount), 2) AS avg_claimamt
FROM claims
JOIN policies USING (policy_id)
GROUP BY 1
ORDER BY 2 DESC;
```

**Results**:
| Policy Type | Average Claim Amount |
|-------------|---------------------|
| Group       | $5,459.74           |
| Individual  | $5,221.90           |
| Family      | $5,166.55           |

**Visualization**:
```
Group      ████████████████████████  5459.74
Individual ██████████████████████    5221.90
Family     █████████████████████     5166.55
```

**Insight**: Group policies have the highest average claim amount, suggesting they may require different pricing or risk management strategies.

---

### 2. Rejected Claims in the Last 12 Months by Type
**Objective**: Analyze which claim types are most frequently rejected to identify potential process improvements.

```sql
SELECT 
    claim_type,
    SUM(CASE WHEN claim_status = 'Rejected' THEN 1 ELSE 0 END) AS cnt
FROM claims
WHERE claim_date >= DATE_SUB(CURDATE(), INTERVAL 1 YEAR)
GROUP BY 1
ORDER BY 2 DESC;
```

**Results**:
| Claim Type      | Rejection Count |
|-----------------|----------------|
| Consultation    | 20             |
| Medication      | 20             |
| Surgery         | 19             |
| Hospitalization | 17             |

**Visualization**:
```
Consultation    ████████████████████████  20
Medication      ████████████████████████  20
Surgery         ███████████████████████   19
Hospitalization ████████████████████      17
```

**Insight**: Consultation and Medication claims have the highest rejection rates, indicating potential issues with documentation requirements or claim submission processes.

---

### 3. Top 10 Policyholders by Total Approved Claims
**Objective**: Identify high-cost customers for focused retention and care management strategies.

```sql
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
```

**Results** (Top 3 shown):
| Policyholder     | Total Approved Claims |
|------------------|----------------------|
| Robert Moore     | $51,827.92           |
| Jeffery Barber   | $36,468.77           |
| Jose Obrien      | $34,196.12           |

**Visualization**:
```
Robert Moore      ████████████████████████  51827.92
Jeffery Barber    ████████████████░░░░░░░░  36468.77
Jose Obrien       ███████████████░░░░░░░░░  34196.12
```

**Insight**: A small group of policyholders account for significant claim costs, warranting specialized attention for fraud detection and care management.

---

### 4. Policyholder Distribution by Age Group and Gender
**Objective**: Understand customer demographics to inform marketing and product development strategies.

```sql
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
```

**Results**:
| Gender | Age Group | Count |
|--------|-----------|-------|
| Female | 18-25     | 12    |
| Male   | 18-25     | 14    |
| Female | 26-35     | 20    |
| Male   | 26-35     | 16    |
| Female | 36-45     | 15    |
| Male   | 36-45     | 20    |
| Female | 46-60     | 16    |
| Male   | 46-60     | 26    |
| Female | 60+       | 33    |
| Male   | 60+       | 28    |

**Visualization**:
```
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
```

**Insight**: The customer base skews older, with the 60+ age group representing the largest segment, particularly females.

---

### 5. Active Policies with No Claims in Last Year
**Objective**: Identify policies with low utilization that may represent profitable customer segments.

```sql
SELECT COUNT(*)
FROM (SELECT * FROM policies WHERE status = 'Active') po
LEFT JOIN policyholders ph USING (policyholder_id)
WHERE NOT EXISTS (
    SELECT 1
    FROM claims c
    WHERE c.policy_id = po.policy_id
      AND c.claim_date >= DATE_SUB(CURDATE(), INTERVAL 1 YEAR)
);
```

**Result**: 76 active policies had no claims in the past 12 months.

**Insight**: These 76 policyholders represent a low-utilization, potentially profitable segment that might benefit from wellness programs or loyalty incentives.

---

### 6. Claim Approval Rate by Claim Type
**Objective**: Measure processing effectiveness across different claim types.

```sql
SELECT 
    claim_type,
    COUNT(*) / (SELECT COUNT(*) FROM claims c WHERE c.claim_type = cs.claim_type) * 100 AS approval_rate
FROM claims cs
WHERE claim_status = 'Approved'
GROUP BY claim_type;
```

**Results**:
| Claim Type      | Approval Rate |
|-----------------|--------------|
| Consultation    | 37.11%       |
| Hospitalization | 35.43%       |
| Medication      | 34.78%       |
| Surgery         | 30.72%       |

**Visualization**:
```
Consultation    ██████████████████████  37.11%
Hospitalization ████████████████████    35.43%
Medication      ████████████████████    34.78%
Surgery         ██████████████████      30.72%
```

**Insight**: Consultation claims have the highest approval rate, while Surgery claims have the lowest, suggesting possible complexities in surgical claim processing.

---

### 7. Policy Status Analysis by Policy Type
**Objective**: Understand policy lifecycle patterns across different product types.

```sql
SELECT 
    policy_type,
    ROUND(100 * SUM(CASE WHEN status = 'Expired' THEN 1 ELSE 0 END) / COUNT(*), 2) AS expired_pct,
    ROUND(100 * SUM(CASE WHEN status = 'Cancelled' THEN 1 ELSE 0 END) / COUNT(*), 2) AS cancelled_pct,
    ROUND(AVG(premium_amount), 2) AS avg_premium
FROM policies
GROUP BY policy_type;
```

**Results**:
| Policy Type | Expired % | Cancelled % | Average Premium |
|-------------|-----------|-------------|----------------|
| Family      | 36.11%    | 27.08%      | $2,969.18      |
| Group       | 32.28%    | 35.43%      | $2,941.70      |
| Individual  | 34.11%    | 27.13%      | $3,024.83      |

**Visualization**:
```
Family
  Expired (%)   ████████████████████ 36.11
  Cancelled (%) ████████████████     27.08
Group
  Expired (%)   █████████████████    32.28
  Cancelled (%) ██████████████████   35.43
Individual
  Expired (%)   ██████████████████   34.11
  Cancelled (%) ████████████████     27.13
```

**Insight**: Group policies show the highest cancellation rate (35.43%), suggesting potential issues with group product satisfaction or retention.

---

### 8. Average Days from Policy Start to First Claim
**Objective**: Measure the timing of initial claims to understand customer behavior patterns.

```sql
WITH cte AS (
    SELECT policy_id, MIN(claim_date) AS first_claim
    FROM claims
    GROUP BY policy_id
)
SELECT 
    ROUND(AVG(TIMESTAMPDIFF(DAY, start_date, first_claim)), 2) AS avg_days_to_first_claim
FROM policies
JOIN cte USING (policy_id);
```

**Result**: 141.57 days (approximately 4.65 months)

**Insight**: The average time to first claim suggests most customers utilize their policies within the first five months of coverage.

---

### 9. Premium-to-Claim Payout Ratio by Policy Type
**Objective**: Measure profitability across different policy types.

```sql
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
```

---

### 10. Policyholders with Multiple Policies
**Objective**: Identify customers with multiple policies to understand cross-selling opportunities and customer value.

```sql
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
```

## Key Takeaways

1. **Product Performance**: Group policies generate the highest average claims ($5,459.74) but also show the highest cancellation rates (35.43%)
2. **Claims Processing**: Consultation and Medication claims have the highest rejection rates (20 each), indicating potential process improvements needed
3. **Customer Segmentation**: A small group of policyholders account for significant claim costs, warranting specialized attention
4. **Demographics**: The customer base skews older, with the 60+ age group representing the largest segment
5. **Utilization Patterns**: 76 active policies had no claims in the past year, representing a potentially profitable segment
6. **Operational Metrics**: First claims typically occur around 141.57 days after policy inception

## Recommendations

1. **Review Group Policy Pricing**: Given the high claim amounts and cancellation rates, consider risk-based pricing adjustments
2. **Claims Process Optimization**: Focus on improving documentation requirements for Consultation and Medication claims to reduce rejections
3. **High-Value Customer Programs**: Develop specialized retention strategies for policyholders with high claim amounts
4. **Senior-Focused Products**: Leverage the older demographic profile to develop targeted products and services
5. **Wellness Programs**: Engage the low-utilization segment with preventive care programs to maintain their health and retention

---
