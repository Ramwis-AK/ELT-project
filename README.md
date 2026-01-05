ELT proces Facebook Ads Analytics v Snowflake
0. Základný prehľad projektu
Tento projekt sa zameriava na návrh a implementáciu end-to-end ELT (Extract – Load – Transform) pipeline v cloudovom dátovom sklade Snowflake. Cieľom je spracovať marketingové dáta z platformy Facebook Ads, transformovať ich do dimenzionálneho modelu (Star Schema) a pripraviť ich na analytické a reportingové využitie.
Projekt simuluje reálny dátovo-inžiniersky a analytický workflow používaný v marketingových a analytických tímoch.

1. Úvod a zdôvodnenie výberu datasetu
Rozhodli sme sa pre tento dataset, pretože sa obaja pohybujeme vo svete digitálneho marketingu a práca s dátami z reklamných platforiem je pre nás prirodzená a prakticky využiteľná.
Dataset z Facebook Ads považujeme za adekvátny najmä z týchto dôvodov:<br>
•	Biznis relevancia – Facebook Ads patria medzi najpoužívanejšie reklamné platformy. Dáta umožňujú analyzovať výkonnosť kampaní, efektívnosť rozpočtov a správanie publika.<br>
•	Reálna komplexnosť dát – dáta obsahujú viacero úrovní (account, campaign, adset, ad), časový rozmer a kombináciu numerických aj kategorizovaných metrík.<br>
•	Vhodnosť pre dimenzionálne modelovanie – štruktúra dát je ideálna pre návrh Star Schema modelu.<br>
•	Prenositeľnosť metodiky – rovnaký prístup je možné aplikovať aj na iné reklamné platformy (Google Ads, TikTok Ads, LinkedIn Ads).<br>
•	Dostupnosť dát – dataset je verejne dostupný prostredníctvom Snowflake Marketplace, čo umožňuje reprodukovateľnosť riešenia.<br>

2. Zdrojové dáta
2.1 Zdroj datasetu<br>
•	Platforma: Snowflake Marketplace<br>
•	Dataset: AD_DATA_FUSION.FACEBOOK_ADS<br>
•	Primárna tabuľka: ADS_INSIGHTS<br>
<img width="849" height="665" alt="image" src="https://github.com/user-attachments/assets/757b8217-1032-4a2a-8f71-c7dc81f82e18" />
<img width="891" height="326" alt="image" src="https://github.com/user-attachments/assets/de3dda1b-33ba-4906-af4b-eb883d20a0e9" />

2.2 Štruktúra zdrojových dát
Zdrojové dáta obsahujú denné metriky reklamných kampaní a sú normalizované do viacerých entít.
Tabuľka	Popis
ADS_INSIGHTS	Denné výkonnostné metriky (impressions, clicks, spend, CTR, CPC, CPM)
CAMPAIGNS	Metadata o kampaniach
ADSETS	Skupiny reklám s targetingom
ADS	Jednotlivé reklamné kreatívy
ACCOUNTS	Facebook Business účty
<img width="851" height="556" alt="image" src="https://github.com/user-attachments/assets/ea8b2220-6867-49a8-ac41-f0206b4648ee" />

2.3 Kľúčové metriky<br>
•	IMPRESSIONS – počet zobrazení reklamy<br>
•	CLICKS – počet kliknutí<br>
•	SPEND – výdavky na reklamu (USD)<br>
•	REACH – počet unikátnych používateľov<br>
•	FREQUENCY – priemerný počet zobrazení na používateľa<br>
•	CTR – Click Through Rate<br>
•	CPC – Cost Per Click<br>
•	CPM – Cost Per Mille<br>
Tieto metriky predstavujú základ pre hodnotenie efektívnosti marketingových kampaní.

3. Ciele analýzy<br>
Hlavné analytické ciele projektu:<br>
    1.	Identifikovať najefektívnejšie kampane a reklamy<br>
    2.	Porovnať nákladovosť kampaní (CPC, CPM)<br>
    3.	Analyzovať trendy výkonnosti v čase<br>
    4.	Detegovať sezónne vzory a anomálie<br>
    5.	Pripraviť dáta pre dashboardy a reporting

4. ELT architektúra<br>
Projekt je postavený na princípe ELT, kde:<br>
•	Extract & Load prebieha priamo zo Snowflake Marketplace do RAW vrstvy<br>
•	Transformácie sú realizované v Snowflake pomocou SQL<br>
•	Analytická vrstva je postavená na dimenzionálnom modeli<br>

5. RAW vrstva (Extract & Load)
5.1 Nastavenie prostredia
```sql
USE ROLE TRAINING_ROLE;
USE WAREHOUSE LYNX_WH;
USE DATABASE LR_PROJECT_DB;
CREATE SCHEMA IF NOT EXISTS RAW;
CREATE SCHEMA IF NOT EXISTS ANALYTICS;
```
5.2 Overenie dostupnosti a štruktúry dát
```sql
SHOW DATABASES;
SELECT COUNT(*) FROM AD_DATA_FUSION.FACEBOOK_ADS.ADS_INSIGHTS;
SELECT *
FROM AD_DATA_FUSION.FACEBOOK_ADS.ADS_INSIGHTS
LIMIT 5;
```
5.3 Načítanie dát do staging tabuľky
```sql
CREATE OR REPLACE TABLE RAW.STG_FB_ADS_INSIGHTS AS
SELECT
CAST(DATE_START AS DATE) AS ad_date,
ACCOUNT_ID,
ACCOUNT_NAME,
CAMPAIGN_ID,
CAMPAIGN_NAME,
ADSET_ID,
ADSET_NAME,
AD_ID,
AD_NAME,
OBJECTIVE,
CAST(IMPRESSIONS AS NUMBER(38,0)) AS IMPRESSIONS,
CAST(CLICKS AS NUMBER(38,0)) AS CLICKS,
CAST(SPEND AS NUMBER(38,2)) AS SPEND,
CAST(REACH AS NUMBER(38,0)) AS REACH,
CAST(FREQUENCY AS FLOAT) AS FREQUENCY,
CAST(CTR AS FLOAT) AS CTR,
CAST(CPM AS FLOAT) AS CPM,
CAST(CPC AS FLOAT) AS CPC,
'FACEBOOK' AS platform,
CURRENT_TIMESTAMP() AS load_timestamp
FROM AD_DATA_FUSION.FACEBOOK_ADS.ADS_INSIGHTS
WHERE
DATE_START IS NOT NULL
AND IMPRESSIONS > 0
AND ACCOUNT_ID IS NOT NULL;
```
5.4 Validácia dát<br>
•	kontrola počtu záznamov<br>
•	kontrola rozsahu dát<br>
•	základná dátová kvalita<br>
```sql
SELECT COUNT(*) AS total_records FROM RAW.STG_FB_ADS_INSIGHTS;
SELECT MIN(ad_date) AS earliest, MAX(ad_date) AS latest FROM RAW.STG_FB_ADS_INSIGHTS;
```

6. TRANSFORM vrstva<br>
Zdrojové dáta obsahovali duplicitné záznamy na úrovni ad_date + ad_id. Tie boli odstránené agregáciou.
6.1 Oprava duplikátov
```sql
SELECT
ad_date,
AD_ID,
COUNT(*) as dup_count
FROM RAW.STG_FB_ADS_INSIGHTS
GROUP BY ad_date, AD_ID
HAVING COUNT(*) > 1;
```
6.2 Deduplikácia
```sql
CREATE OR REPLACE TABLE RAW.STG_FB_ADS_DEDUPLICATED AS
SELECT
ad_date,
ACCOUNT_ID,
ACCOUNT_NAME,
CAMPAIGN_ID,
CAMPAIGN_NAME,
ADSET_ID,
ADSET_NAME,
AD_ID,
AD_NAME,
OBJECTIVE,
ROUND(AVG(IMPRESSIONS)) AS IMPRESSIONS,
ROUND(AVG(CLICKS)) AS CLICKS,
ROUND(AVG(SPEND), 2) AS SPEND,
ROUND(AVG(REACH)) AS REACH,
ROUND(AVG(FREQUENCY), 4) AS FREQUENCY,
ROUND(AVG(CTR), 4) AS CTR,
ROUND(AVG(CPM), 2) AS CPM,
ROUND(AVG(CPC), 4) AS CPC,
platform,
MAX(load_timestamp) AS load_timestamp
FROM RAW.STG_FB_ADS_INSIGHTS
GROUP BY
ad_date, ACCOUNT_ID, ACCOUNT_NAME, CAMPAIGN_ID, CAMPAIGN_NAME,
ADSET_ID, ADSET_NAME, AD_ID, AD_NAME, OBJECTIVE, platform;
```
6.3 Validácia deduplikácie
```sql
SELECT COUNT(*) FROM RAW.STG_FB_ADS_DEDUPLICATED;
```
6.4 Detekcia anomálií<br>
Identifikované extrémne hodnoty CPC a CPM, ktoré môžu signalizovať chyby alebo neštandardné správanie kampaní.<br>
```sql
SELECT
ad_date,
CAMPAIGN_NAME,
CPC,
CPM,
SPEND,
IMPRESSIONS
FROM RAW.STG_FB_ADS_DEDUPLICATED
WHERE CPC > 100 OR CPM > 50 OR CPC < 0.001
ORDER BY CPC DESC;
```
7. Dimenzionálny model – Star Schema
7.1 Návrh Star Schema<br>
Model pozostáva z jednej faktovej tabuľky a viacerých dimenzií.<br>
<img width="945" height="653" alt="image" src="https://github.com/user-attachments/assets/d8d3de86-2a3a-4afc-abd3-0a292975d475" />

7.2 Dimenzie<br>
•	DIM_ACCOUNT – reklamné účty (SCD Type 1 - bez histórie)<br>
•	DIM_CAMPAIGN – kampane (SCD Type 2 - s históriou)<br>
•	DIM_ADSET – skupiny reklám (SCD Type 1)<br>
•	DIM_AD – jednotlivé reklamy (SCD Type 2)<br>
•	DIM_DATE – časová dimenzia<br>

```sql
CREATE OR REPLACE TABLE ANALYTICS.DIM_ACCOUNT AS SELECT DISTINCT
ACCOUNT_ID,
ACCOUNT_NAME,
'USA' AS country,
CURRENT_DATE AS created_date
FROM RAW.STG_FB_ADS_DEDUPLICATED
WHERE ACCOUNT_ID IS NOT NULL;

CREATE OR REPLACE TABLE ANALYTICS.DIM_ADSET AS SELECT DISTINCT
ADSET_ID,
ADSET_NAME,
CAMPAIGN_ID,
'ACTIVE' AS status,
CURRENT_DATE AS created_date
FROM RAW.STG_FB_ADS_DEDUPLICATED
WHERE ADSET_ID IS NOT NULL;

CREATE OR REPLACE TABLE ANALYTICS.DIM_CAMPAIGN AS SELECT
CAMPAIGN_ID,
CAMPAIGN_NAME,
OBJECTIVE,
'ACTIVE' AS is_active,
MIN(ad_date) AS created_date,
MIN(ad_date) AS valid_form,
TO_DATE('2099-12-31') as valid_to,
TRUE AS is_current
FROM RAW.STG_FB_ADS_DEDUPLICATED
GROUP BY CAMPAIGN_ID, CAMPAIGN_NAME, OBJECTIVE;

CREATE OR REPLACE TABLE ANALYTICS.DIM_AD AS SELECT
AD_ID,
AD_NAME,
ADSET_ID,
MIN(ad_date) AS created_date,
MIN(ad_date) AS valid_form,
TO_DATE('2099-12-31') as valid_to,
TRUE AS is_current
FROM RAW.STG_FB_ADS_DEDUPLICATED
GROUP BY AD_ID, AD_NAME, ADSET_ID;

CREATE OR REPLACE TABLE ANALYTICS.DIM_DATE AS
WITH date_range AS (
SELECT DATEADD(DAY, SEQ4(), DATE('2020-01-01')) AS date
FROM TABLE(GENERATOR(ROWCOUNT => 2000))
)
SELECT
date,
YEAR(date) AS year,
MONTH(date) AS month,
DAY(date) AS day,
WEEKOFYEAR(date) AS week,
QUARTER(date) AS quarter,
DAYOFWEEK(date) AS day_of_week,
DAYOFWEEK(date) IN (6, 7) AS is_weekend,
TO_CHAR(date, 'MMMM') AS month_name
FROM date_range
WHERE date BETWEEN DATE('2020-01-01') AND DATE('2024-12-31')
ORDER BY date;
```


7.3 Faktová tabuľka<br>
FACT_AD_DAILY obsahuje denné metriky a pokročilé výpočty pomocou window functions.<br>
Použité window functions:<br>
•	LAG – zmena CPC deň po dni<br>
•	RANK – poradie reklám<br>
•	SUM OVER – kumulatívne výdavky<br>
•	AVG OVER – 7-dňový priemer<br>
```sql
CREATE OR REPLACE TABLE ANALYTICS.FACT_AD_DAILY AS
WITH enriched AS (
SELECT
ROW_NUMBER() OVER (ORDER BY s.ad_date, s.AD_ID) AS fact_id,
s.ad_date,
s.CAMPAIGN_ID,
s.ADSET_ID,
s.ACCOUNT_ID,
s.AD_ID,
s.IMPRESSIONS,
s.CLICKS,
s.SPEND,
s.REACH,
s.FREQUENCY,
s.CTR,
s.CPM,
s.CPC,
LAG(s.CPC, 1) OVER (PARTITION BY s.AD_ID ORDER BY s.ad_date) AS cpc_prev_day,
ROW_NUMBER() OVER (PARTITION BY s.ad_date ORDER BY s.CPC ASC) AS daily_rank_by_cpc_asc,
RANK() OVER (PARTITION BY s.AD_ID ORDER BY s.ad_date DESC ROWS BETWEEN CURRENT ROW AND 6 FOLLOWING) AS rank_7d_window,
SUM(s.SPEND) OVER (PARTITION BY s.AD_ID ORDER BY s.ad_date) AS cumulative_spend_per_ad,
ROUND(AVG(s.CTR) OVER (PARTITION BY s.AD_ID ORDER BY s.ad_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW), 4) AS ctr_7d_avg
FROM RAW.STG_FB_ADS_DEDUPLICATED s
)
SELECT * FROM enriched;
```

8. Vizualizácie a analytické výstupy<br>
Projekt obsahuje viacero analytických pohľadov pripravených na vizualizáciu.<br>
Každá vizualizácia obsahuje:<br>
•	SQL dotaz<br>
•	popis<br>
•	business interpretáciu<br>

VIZ #1 – Trend CPC v čase
```sql
SELECT
    CONCAT(d.year, '-', LPAD(d.month, 2, '0')) AS year_month, d.month_name,
    ROUND(AVG(f.CPC), 4) AS avg_cpc,
    ROUND(MIN(f.CPC), 4) AS min_cpc,
    ROUND(MAX(f.CPC), 4) AS max_cpc,
    SUM(f.CLICKS) AS total_clicks,
    ROUND(SUM(f.SPEND), 2) AS monthly_spend,
    COUNT(DISTINCT f.AD_ID) AS unique_ads
FROM ANALYTICS.FACT_AD_DAILY f
JOIN ANALYTICS.DIM_DATE d ON f.ad_date = d.date
GROUP BY d.year, d.month, d.month_name
ORDER BY d.year, d.month;
```
<img width="945" height="453" alt="image" src="https://github.com/user-attachments/assets/2946c049-e269-411d-a03f-66edc7ec3c4e" />


VIZ #2 – Top kampane podľa spendu
```sql
SELECT TOP 10
    c.CAMPAIGN_NAME,
    c.OBJECTIVE,
    ROUND(SUM(f.SPEND), 2) AS total_spend,
    SUM(f.IMPRESSIONS) AS total_impressions,
    SUM(f.CLICKS) AS total_clicks,
    ROUND(AVG(f.CTR), 4) AS avg_ctr_percent,
    ROUND(AVG(f.CPC), 4) AS avg_cpc,
    ROUND(AVG(f.CPM), 4) AS avg_cpm,
    COUNT(DISTINCT f.ad_date) AS days_active,
    COUNT(DISTINCT f.AD_ID) AS num_ads
FROM ANALYTICS.FACT_AD_DAILY f
JOIN ANALYTICS.DIM_CAMPAIGN c ON f.CAMPAIGN_ID = c.campaign_id
GROUP BY c.campaign_id, c.campaign_name, c.objective
ORDER BY total_spend DESC;
```
<img width="945" height="551" alt="image" src="https://github.com/user-attachments/assets/22d34b27-2e42-4fef-bb47-5330854929a7" />


VIZ #3: Efektívnosť kampaní (Spend vs. Reach) - SCATTER PLOT
```sql
SELECT
    c.CAMPAIGN_NAME,
    ROUND(SUM(f.SPEND), 2) AS total_spend,
    SUM(f.REACH) AS total_reach,
    ROUND(AVG(f.CTR), 4) AS avg_ctr,
    SUM(f.CLICKS) AS total_clicks,
    COUNT(DISTINCT f.AD_ID) AS num_ads,
    ROUND(SUM(f.SPEND) / NULLIF(SUM(f.REACH), 0), 4) AS spend_per_reach,
    ROUND(SUM(f.SPEND) / NULLIF(SUM(f.CLICKS), 0), 4) AS spend_per_click,
FROM ANALYTICS.FACT_AD_DAILY f
JOIN ANALYTICS.DIM_CAMPAIGN c ON f.campaign_id = c.campaign_id
GROUP BY c.campaign_id, c.CAMPAIGN_NAME
HAVING SUM(f.IMPRESSIONS) > 50
ORDER BY total_spend DESC;
```
<img width="945" height="539" alt="image" src="https://github.com/user-attachments/assets/4091b758-7053-4e77-8d88-8413a78d0f71" />


VIZ #4: CTR podľa cieľov kampánie - BAR CHART
```sql
SELECT
    c.OBJECTIVE,
    COUNT(DISTINCT f.AD_ID) AS num_ads,
        ROUND(AVG(f.CTR), 4) AS avg_ctr,
    ROUND(MIN(f.CTR), 4) AS min_ctr,
    ROUND(MAX(f.CTR), 4) AS max_ctr,
    SUM(f.IMPRESSIONS) AS total_impressions,
    SUM(f.CLICKS) AS total_clicks,
    ROUND(SUM(f.SPEND), 2) AS total_spend,
    ROUND(AVG(f.CPC), 4) AS avg_cpc
FROM ANALYTICS.FACT_AD_DAILY f
JOIN ANALYTICS.DIM_CAMPAIGN c ON f.CAMPAIGN_ID = c.CAMPAIGN_ID
WHERE c.objective IS NOT NULL
GROUP BY c.objective
ORDER BY avg_ctr DESC;
```
<img width="945" height="550" alt="image" src="https://github.com/user-attachments/assets/a9496d71-3da4-4119-9370-9009d7b175cc" />


VIZ #5: Top annonce podľa ROI (Spend vs. Engagement) – TABLE
```sql
SELECT TOP 15
    a.AD_NAME,
    s.ADSET_NAME,
    c.CAMPAIGN_NAME,
    COUNT(DISTINCT f.ad_date) AS days_active,
    ROUND(SUM(f.SPEND), 2) AS impressions,
    SUM(f.IMPRESSIONS) AS impressions,
    SUM(f.CLICKS) AS clicks,
    SUM(f.REACH) AS reach,
    ROUND(AVG(f.CTR), 4) AS avg_ctr,
    ROUND(AVG(f.CPC), 4) AS avg_cpc,
    ROUND(AVG(f.CPM), 2) AS avg_cpm,
    ROUND(SUM(f.SPEND) / NULLIF(SUM(f.CLICKS), 0), 4) AS cost_per_click,
    RANK() OVER (ORDER BY AVG(f.CPC) ASC) AS efficiency_rank
FROM ANALYTICS.FACT_AD_DAILY f
JOIN ANALYTICS.DIM_AD a ON f.AD_ID = a.AD_ID
JOIN ANALYTICS.DIM_ADSET s ON a.ADSET_ID = s.ADSET_ID
JOIN ANALYTICS.DIM_CAMPAIGN c ON f.CAMPAIGN_ID = c.CAMPAIGN_ID
GROUP BY a.AD_ID, a.AD_NAME, s.ADSET_NAME, c.CAMPAIGN_NAME
HAVING SUM(f.IMPRESSIONS) > 100
ORDER BY avg_cpc ASC;
```
<img width="945" height="557" alt="image" src="https://github.com/user-attachments/assets/e8c52809-4eac-46db-93af-65025fdfff28" />


VIZ #6: Sezónnosť - Výdavky podľa dňa týždňa – HEATMAP
```sql
SELECT
    d.day_of_week,
    CASE
        WHEN d.day_of_week = 1 THEN 'Pondelok'
        WHEN d.day_of_week = 2 THEN 'Utorok'
        WHEN d.day_of_week = 3 THEN 'Streda'
        WHEN d.day_of_week = 4 THEN 'Štvrtok'
        WHEN d.day_of_week = 5 THEN 'Piatok'
        WHEN d.day_of_week = 6 THEN 'Sobota'
        WHEN d.day_of_week = 7 THEN 'Nedeľa'
    END AS day_name,
    COUNT(*) AS num_days,
    ROUND(AVG(f.SPEND), 2) AS avg_daily_spend,
    ROUND(AVG(f.CTR), 4) AS avg_ctr,
    ROUND(AVG(f.CPC), 4) AS avg_cpc,
    SUM(f.CLICKS) AS total_clicks,
    SUM(f.IMPRESSIONS) AS total_impressions
FROM ANALYTICS.FACT_AD_DAILY f
JOIN ANALYTICS.DIM_DATE d ON f.ad_date = d.date
GROUP BY d.day_of_week
ORDER BY d.day_of_week;
```
<img width="945" height="524" alt="image" src="https://github.com/user-attachments/assets/5da17558-71d6-4ddb-8bf1-e7223763a934" />

 
VIZ #7: Kumulatívne výdavky podľa kampánie (WINDOW FUNCTION) - AREA CHART
```sql
SELECT
    d.date,
    d.month_name,
    c.CAMPAIGN_NAME,
    SUM(f.SPEND) AS daily_spend,
    SUM(SUM(f.SPEND)) OVER (PARTITION BY c.CAMPAIGN_ID ORDER BY d.date) AS cumulative_spend,
FROM ANALYTICS.FACT_AD_DAILY f
JOIN ANALYTICS.DIM_DATE d ON f.ad_date = d.date
JOIN ANALYTICS.DIM_CAMPAIGN c ON f.CAMPAIGN_ID = c.CAMPAIGN_ID
GROUP BY d.date, d.month_name, c.CAMPAIGN_ID, c.CAMPAIGN_NAME
ORDER BY c.CAMPAIGN_ID, d.date;
DESC TABLE ANALYTICS.FACT_AD_DAILY;
```
<img width="945" height="433" alt="image" src="https://github.com/user-attachments/assets/d97bf3ff-a1b6-45ce-a10b-dc091ac6ae5b" />


VIZ #8: Porovnanie CPC v čase s LAG window function - LINE CHART
```sql
SELECT
    f.ad_date,
    f.AD_ID,
    ROUND(f.CPC, 4) AS cpc_today,
    ROUND(f.cpc_prev_day, 4) AS cpc_yesterday,
    CASE
        WHEN f.cpc_prev_day IS NOT NULL THEN 0
        ELSE ROUND(((f.CPC - f.cpc_prev_day) / f.cpc_prev_day) *100, 2)
    END AS percent_change,
    f.IMPRESSIONS,
    f.CLICKS,
    ROUND(f.SPEND, 2) AS spend
FROM ANALYTICS.FACT_AD_DAILY f
WHERE f.cpc_prev_day IS NOT NULL
ORDER BY f.ad_id, f.ad_date
LIMIT 50;
```
<img width="945" height="456" alt="image" src="https://github.com/user-attachments/assets/b6280688-018d-49b9-9467-221ec1fc8386" />


VIZ #9: Ranking annoncí v 7-dňovom okne (RANK window function) - DETAILED TABLE
```sql
SELECT
    f.AD_ID,
    f.ad_date,
    EXTRACT(WEEK FROM f.ad_date) AS week_number,
    ROUND(f.CPC, 4) AS cpc,
    f.rank_7d_window AS rank_in_7day_window,
    f.CLICKS,
    f.IMPRESSIONS,
    CASE
        WHEN f.rank_7d_window = 1 THEN 'Best Performer'
        WHEN f.rank_7d_window <= 3 THEN 'TOP 3'
        WHEN f.rank_7d_window <= 5 THEN 'GOOD'
        ELSE 'Needs Optimization'
    END AS performance_tier
FROM ANALYTICS.FACT_AD_DAILY f
ORDER BY f.ad_id, f.ad_date, f.rank_7d_window
LIMIT 100;
```
<img width="945" height="430" alt="image" src="https://github.com/user-attachments/assets/34eb0dee-1321-47be-ba22-fe55d1d74c05" />
<img width="945" height="441" alt="image" src="https://github.com/user-attachments/assets/bf271d7b-0be8-4cc5-8778-55b5c02fda37" />


VIZ #10: Performance Dashboard - Overview Metrics - KPI CARD
```sql
SELECT
    COUNT(DISTINCT f.fact_id) AS total_ad_days,
    COUNT(DISTINCT f.CAMPAIGN_ID) AS num_campaigns,
    COUNT(DISTINCT f.AD_ID) AS num_unique_ads,
    COUNT(DISTINCT f.ad_date) AS days_with_data,
    ROUND(SUM(f.SPEND), 2) AS total_spend_usd,
    SUM(f.IMPRESSIONS) AS total_impressions,
    SUM(f.CLICKS) AS total_clicks,
    SUM(f.REACH) AS total_reach,
    ROUND(AVG(f.CTR), 4) AS avg_ctr_percent,
    ROUND(AVG(f.CPC), 4) AS avg_cpc,
    ROUND(AVG(f.CPM), 2) AS avg_cpm,
    ROUND(AVG(f.FREQUENCY), 2) AS avg_frequency,
    ROUND(SUM(f.SPEND) / NULLIF(SUM(f.REACH), 0), 4) AS spend_per_person,
    MIN(f.ad_date) AS campaign_start,
    MAX(f.ad_date) AS campaign_end,
    DATEDIFF(DAY, MIN(f.ad_date), MAX(f.ad_date)) + 1 AS campaign_duration_days
FROM ANALYTICS.FACT_AD_DAILY f;
```
<img width="945" height="187" alt="image" src="https://github.com/user-attachments/assets/301dd7bd-21a1-49a3-b92d-7bfb84a5550b" />

9. Performance Dashboard<br>
Dashboard poskytuje high-level prehľad pre manažment:<br>
•	celkové výdavky<br>
•	impressions, clicks, reach<br>
•	priemerné CPC, CTR, CPM<br>
•	trvanie kampaní<br>

10. Kľúčové insights<br>
•	Star Schema výrazne zjednodušuje analytické dotazy<br>
•	Window functions umožňujú pokročilé analýzy bez externých nástrojov<br>
•	Dáta sú pripravené na multi-platform rozšírenie<br>

11. Technické nástroje<br>
•	Snowflake<br>
•	Snowflake Marketplace<br>
•	SQL<br>
•	Vizualizačný nástroj (Power BI / Tableau / Looker)<br>

12. Možné rozšírenia projektu<br>
•	Incrementálny load<br>
•	Alerting na anomálie<br>
•	Prediktívne modelovanie<br>
•	Integrácia ďalších reklamných platforiem<br>

Autori<br>
•	Alexander Krobot<br>
•	Filip Samko<br>
Dátum: Január 2026<br>
Verzia: 1.0<br>

Referencie<br>
•	Snowflake Documentation<br>
•	Ralph Kimball – The Data Warehouse Toolkit<br>
•	Window Functions in SQL<br>
