# Airport-Performance
This project investigates why aircraft experience prolonged taxi‑out times at major German airports. By examining arrival delays, categorized delay factors, and operational performance, this analysis highlights which factors act as genuine drivers of taxi‑out inefficiency.

* **Study Period**: 2018–2024.
* **Airports**: Frankfurt (EDDF), Munich (EDDM), Düsseldorf (EDDL), Cologne‑Bonn (EDDK), Leipzig‑Halle (EDDP).

## 👥 Contributor

**Vy Hoang** — Data Analyst

**Project Type**: Personal business case analysis (non‑commercial)

**Tools:** BigQuery, R, Tableau

**Tag**: Statistical Analysis, Visualizaion, Aviation Analytics 

**Primary Questions:**

* Do arrival delays meaningfully influence taxi‑out time?
* Which delay categories correlate most with taxi‑out additional minutes?
* Where can airports prioritize operational improvements?

## Why Taxi-out Time Delays Matter?

* **Average Taxi‑Out Time (ECAC)**: 13.8 minutes at large/very large airports (2018).
* **Economic Cost**: ~€182 per minute of ground delay.
* **Environmental Impact**: ~25.8 kg fuel burned per minute → 80+ kg CO₂.
---

## Key Findings

1. **Arrival delay is not a strong predictor of taxi‑out performance** (R² ≈ 0.076).
2. **Weather** shows the strongest positive correlation with taxi‑out additional time.
3. **Disruption delays** have a moderate effect, creating localized bottlenecks.
4. **Capacity, staffing, and event delays** show weak or no meaningful correlation.
5. Distributions are right‑skewed → a few severe delay months heavily influence the mean.

---

## 📁 Repository Structure

```
├─ README.md
├─ data/
│  ├─ raw/
│  └─ processed/
├─ sql/
│  └─ queries.sql
├─ r/
│  ├─ 01_missing_values_handling.R
│  ├─ 02_analysis.R
│  └─ 03_reporting.R
├─ notebooks/
│  └─ exploratory.ipynb
├─ dashboards/
│  └─ tableau/
├─ figures/
├─ .github/
│  └─ workflows/ci.yml
└─ LICENSE
```

---

## Data Pipeline Summary

### **1. Data Collection**

* Source: EUROCONTROL Performance Data Portal (https://ansperformance.eu/data/)
* Datasets used:
        ** Airport arrival ATFM delays (with post ops adjustment)
        ** Taxi-out additional time
---

### **2. Data Cleaning & Transformation (BigQuery)**

Data preparation integrated arrival metrics, taxi‑out metrics, and grouped delay reasons. Key fields constructed:

#### **Arrival Metrics**

```sql
arr.FLT_ARR_1 AS arrival_flights,
arr.DLY_APT_ARR_1 AS total_arrival_delay_minutes,
(arr.DLY_APT_ARR_1 / NULLIF(arr.FLT_ARR_1, 0)) AS avg_arrival_delay_per_flight,
```

#### **Taxi‑Out Metrics**

```sql
taxi.TOTAL_REF_NB_FL AS departure_flights,
taxi.TOTAL_ADD_TIME_MIN AS total_taxiout_additional_minutes,
(taxi.TOTAL_ADD_TIME_MIN / NULLIF(taxi.TOTAL_REF_NB_FL, 0)) AS avg_taxiout_additional_time,
```

#### **Grouped Delay Reason Categories**

```sql
(DLY_APT_ARR_A_1 + DLY_APT_ARR_E_1 + DLY_APT_ARR_I_1 + 
 DLY_APT_ARR_N_1 + DLY_APT_ARR_O_1 + DLY_APT_ARR_T_1) AS disruption_delay,

(DLY_APT_ARR_C_1 + DLY_APT_ARR_G_1 + DLY_APT_ARR_M_1 + 
 DLY_APT_ARR_R_1 + DLY_APT_ARR_V_1) AS capacity_delay,

(DLY_APT_ARR_D_1 + DLY_APT_ARR_W_1) AS weather_delay,

DLY_APT_ARR_P_1 AS event_delay,
DLY_APT_ARR_S_1 AS staffing_delay,
```

#### **Table Join Logic**

```sql
FROM `...airport_arrival_delay` AS arr
JOIN `...taxi_out` AS taxi
  ON arr.APT_ICAO = taxi.APT_ICAO
 AND arr.YEAR = taxi.YEAR
 AND arr.MONTH_NUM = taxi.MONTH_NUM
WHERE arr.STATE_NAME = 'Germany'
  AND arr.YEAR BETWEEN 2018 AND 2024
ORDER BY arr.APT_ICAO, arr.YEAR, arr.MONTH_NUM;
```

---

### **3. Data Quality Assessment (BigQuery)**

Assessed % completeness across airports to filter reliable datasets.

```sql
SELECT arr.APT_ICAO, arr.APT_NAME,
       COUNT(*) AS total_months,
       COUNT(DLY_APT_ARR_1) AS months_with_data,
       ROUND(COUNT(DLY_APT_ARR_1) * 100.0 / COUNT(*), 2) AS pct_months_with_data
FROM `...airport_arrival_delay` AS arr
WHERE STATE_NAME = 'Germany'
  AND YEAR BETWEEN 2018 AND 2024
GROUP BY apt_icao, apt_name
ORDER BY pct_months_with_data DESC;
```
| | APT_ICAO | APT_NAME           | total_months | months_with_data | pct_months_with_data |
|----------|--------------------|---------------|-------------------|------------------------|
| EDDK     | Cologne-Bonn       | 84            | 78                | 92.86                 |
| EDDF     | Frankfurt          | 84            | 73                | 86.90                 |
| EDDP     | Leipzig-Halle      | 84            | 73                | 86.90                 |
| EDDM     | Munich             | 84            | 66                | 78.57                 |
| EDDL     | Düsseldorf         | 84            | 64                | 76.19                 |
| EDDT     | Berlin/Tegel       | 35            | 26                | 74.29                 |
| EDDH     | Hamburg            | 84            | 49                | 58.33                 |
| EDDS     | Stuttgart          | 84            | 39                | 46.43                 |
| EDDB     | Berlin/Schoenefeld | 84            | 31                | 36.90                 |

**Selected airports (>70% completeness):**

* Cologne‑Bonn (EDDK)
* Frankfurt (EDDF)
* Leipzig‑Halle (EDDP)
* Munich (EDDM)
* Düsseldorf (EDDL)

---

### **4. Missing Value Treatment (R)**

Median imputation performed **per airport** to preserve local operational characteristics.

```r
df_imputed <- df %>%
  group_by(APT_ICAO) %>%
  mutate(
    median_avg_delay = median(avg_arrival_delay_per_flight, na.rm = TRUE),
    avg_arrival_delay_per_flight_imputed = ifelse(
      is.na(avg_arrival_delay_per_flight), median_avg_delay, avg_arrival_delay_per_flight
    ),
    total_arrival_delay_minutes_imputed = ifelse(
      is.na(total_arrival_delay_minutes),
      avg_arrival_delay_per_flight_imputed * arrival_flights,
      total_arrival_delay_minutes
    )
  ) %>% ungroup()
```

Distribution check confirmed **right‑skewed delay behavior**, driven by a few severe months.

---

### **5. Statistical Analysis (R)**

#### **Arrival Delay vs Taxi‑Out Additional Time**

* Computed overall and per‑airport correlations.

```r
overall_correlation <- cor(
  df_imputed$avg_arrival_delay_per_flight_imputed,
  df_imputed$avg_taxiout_additional_time,
  use = "complete.obs"
)
```

**Airport‑level summary:**

* Düsseldorf → **0.594 (strong)**
* Cologne‑Bonn → **0.365 (moderate)**
* Frankfurt & Munich → **~0.27 (weak)**
* Leipzig‑Halle → **0.053 (very weak)**

#### **Delay Categories vs Taxi‑Out Time**

```r
calculate_correlation <- function(delay_type) {
  complete_cases <- complete.cases(delay_data[[delay_type]], delay_data$avg_taxiout_additional_time)
  cor(delay_data[[delay_type]][complete_cases], delay_data$avg_taxiout_additional_time[complete_cases])
}
```

**Results (R, p‑values):**

| Delay Reason | R      | P‑value |
| ------------ | ------ | ------- |
| Weather      | 0.335  | 1e‑10   |
| Disruption   | 0.116  | 0.029   |
| Capacity     | 0.083  | 0.12    |
| Staffing     | ‑0.042 | 0.43    |
| Event        | ‑0.014 | 0.79    |

Weather emerged as **the only consistently strong and significant driver** across airports.

---

### **6. Data Visualization and Presentation

* - Tableau dashboard: [Interactive dashboard]([https://public.tableau.com/app/profile/vy.hoang7117/viz/AirlinePerformance_17630531546060/Dashboard4](https://public.tableau.com/app/profile/vy.hoang7117/viz/AirlinePerformance_17630531546060/Dashboard4))

* Project Presentation: [YouTube Recording](https://www.youtube.com/watch?v=zTwFryfgfrM&t=1789s) [(29:43 - 39:00)](https://www.youtube.com/watch?v=zTwFryfgfrM&list=PL9ZjLysT1KACdlY6yhErcDCNj1j6LXRz7&index=2)

---

## ▶️ Reproduction Steps

```bash
Rscript r/01_missing_value_handling.R
Rscript r/02_analysis.R
Rscript r/03_reporting.R
```
---

## References

European Organisation for the Safety of Air Navigation. (n.d.). *Taxiing time – EUROCONTROL Standard Inputs for Economic Analyses*. ANS Performance. Retrieved from [https://ansperformance.eu/references/standard-inputs/taxiing-time/](https://ansperformance.eu/references/standard-inputs/taxiing-time/)

European Organisation for the Safety of Air Navigation. (n.d.). *Cost of delay – EUROCONTROL Standard Inputs for Economic Analyses*. ANS Performance. Retrieved from [https://ansperformance.eu/references/standard-inputs/cost-of-delay/](https://ansperformance.eu/references/standard-inputs/cost-of-delay/)

European Organisation for the Safety of Air Navigation. (n.d.). *Rate of fuel burn – EUROCONTROL Standard Inputs for Economic Analyses*. ANS Performance. Retrieved from [https://ansperformance.eu/references/standard-inputs/rate-of-fuel-burn/](https://ansperformance.eu/references/standard-inputs/rate-of-fuel-burn/)

---
**Contact**

**LinkedIn:** [Vy Hoang](https://www.linkedin.com/in/vyhoang-ussh/) | **Email:** vy.hoangphamuyen@gmail.com
