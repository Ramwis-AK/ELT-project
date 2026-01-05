ELT proces Facebook Ads Analytics v Snowflake
0. Základný prehľad projektu
Tento projekt sa zameriava na návrh a implementáciu end-to-end ELT (Extract – Load – Transform) pipeline v cloudovom dátovom sklade Snowflake. Cieľom je spracovať marketingové dáta z platformy Facebook Ads, transformovať ich do dimenzionálneho modelu (Star Schema) a pripraviť ich na analytické a reportingové využitie.
Projekt simuluje reálny dátovo-inžiniersky a analytický workflow používaný v marketingových a analytických tímoch.

1. Úvod a zdôvodnenie výberu datasetu
Rozhodli sme sa pre tento dataset, pretože sa obaja pohybujeme vo svete digitálneho marketingu a práca s dátami z reklamných platforiem je pre nás prirodzená a prakticky využiteľná.
Dataset z Facebook Ads považujeme za adekvátny najmä z týchto dôvodov:
•	Biznis relevancia – Facebook Ads patria medzi najpoužívanejšie reklamné platformy. Dáta umožňujú analyzovať výkonnosť kampaní, efektívnosť rozpočtov a správanie publika.
•	Reálna komplexnosť dát – dáta obsahujú viacero úrovní (account, campaign, adset, ad), časový rozmer a kombináciu numerických aj kategorizovaných metrík.
•	Vhodnosť pre dimenzionálne modelovanie – štruktúra dát je ideálna pre návrh Star Schema modelu.
•	Prenositeľnosť metodiky – rovnaký prístup je možné aplikovať aj na iné reklamné platformy (Google Ads, TikTok Ads, LinkedIn Ads).
•	Dostupnosť dát – dataset je verejne dostupný prostredníctvom Snowflake Marketplace, čo umožňuje reprodukovateľnosť riešenia.

2. Zdrojové dáta
2.1 Zdroj datasetu
•	Platforma: Snowflake Marketplace
•	Dataset: AD_DATA_FUSION.FACEBOOK_ADS
•	Primárna tabuľka: ADS_INSIGHTS


2.2 Štruktúra zdrojových dát
Zdrojové dáta obsahujú denné metriky reklamných kampaní a sú normalizované do viacerých entít.
Tabuľka	Popis
ADS_INSIGHTS	Denné výkonnostné metriky (impressions, clicks, spend, CTR, CPC, CPM)
CAMPAIGNS	Metadata o kampaniach
ADSETS	Skupiny reklám s targetingom
ADS	Jednotlivé reklamné kreatívy
ACCOUNTS	Facebook Business účty


2.3 Kľúčové metriky
•	IMPRESSIONS – počet zobrazení reklamy
•	CLICKS – počet kliknutí
•	SPEND – výdavky na reklamu (USD)
•	REACH – počet unikátnych používateľov
•	FREQUENCY – priemerný počet zobrazení na používateľa
•	CTR – Click Through Rate
•	CPC – Cost Per Click
•	CPM – Cost Per Mille
Tieto metriky predstavujú základ pre hodnotenie efektívnosti marketingových kampaní.

3. Ciele analýzy
Hlavné analytické ciele projektu:
1.	Identifikovať najefektívnejšie kampane a reklamy
2.	Porovnať nákladovosť kampaní (CPC, CPM)
3.	Analyzovať trendy výkonnosti v čase
4.	Detegovať sezónne vzory a anomálie
5.	Pripraviť dáta pre dashboardy a reporting

4. ELT architektúra
Projekt je postavený na princípe ELT, kde:
•	Extract & Load prebieha priamo zo Snowflake Marketplace do RAW vrstvy
•	Transformácie sú realizované v Snowflake pomocou SQL
•	Analytická vrstva je postavená na dimenzionálnom modeli

5. RAW vrstva (Extract & Load)
5.1 Nastavenie prostredia
USE ROLE TRAINING_ROLE;
USE WAREHOUSE LYNX_WH;
USE DATABASE LR_PROJECT_DB;
CREATE SCHEMA IF NOT EXISTS RAW;
CREATE SCHEMA IF NOT EXISTS ANALYTICS;
5.2 Overenie dostupnosti a štruktúry dát
SHOW DATABASES;
SELECT COUNT(*) FROM AD_DATA_FUSION.FACEBOOK_ADS.ADS_INSIGHTS;
SELECT *
FROM AD_DATA_FUSION.FACEBOOK_ADS.ADS_INSIGHTS
LIMIT 5;
5.3 Načítanie dát do staging tabuľky
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
5.2 Validácia dát
•	kontrola počtu záznamov
•	kontrola rozsahu dát
•	základná dátová kvalita
SELECT COUNT(*) AS total_records FROM RAW.STG_FB_ADS_INSIGHTS;
SELECT MIN(ad_date) AS earliest, MAX(ad_date) AS latest FROM RAW.STG_FB_ADS_INSIGHTS;

6. TRANSFORM vrstva
Zdrojové dáta obsahovali duplicitné záznamy na úrovni ad_date + ad_id. Tie boli odstránené agregáciou.
6.1 Oprava duplikátov
SELECT
ad_date,
AD_ID,
COUNT(*) as dup_count
FROM RAW.STG_FB_ADS_INSIGHTS
GROUP BY ad_date, AD_ID
HAVING COUNT(*) > 1;
6.2 Deduplikácia
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
6.3 Validácia deduplikácie
SELECT COUNT(*) FROM RAW.STG_FB_ADS_DEDUPLICATED;
6.4 Detekcia anomálií
Identifikované extrémne hodnoty CPC a CPM, ktoré môžu signalizovať chyby alebo neštandardné správanie kampaní.
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
7. Dimenzionálny model – Star Schema
7.1 Návrh Star Schema
Model pozostáva z jednej faktovej tabuľky a viacerých dimenzií.


7.2 Dimenzie
•	DIM_ACCOUNT – reklamné účty
•	DIM_CAMPAIGN – kampane (SCD-ready)
•	DIM_ADSET – skupiny reklám
•	DIM_AD – jednotlivé reklamy
•	DIM_DATE – časová dimenzia
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

7.3 Faktová tabuľka
FACT_AD_DAILY obsahuje denné metriky a pokročilé výpočty pomocou window functions.
Použité window functions:
•	LAG – zmena CPC deň po dni
•	RANK – poradie reklám
•	SUM OVER – kumulatívne výdavky
•	AVG OVER – 7-dňový priemer
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
<img width="985" height="695" alt="image" src="https://github.com/user-attachments/assets/f118aacd-5bb1-4a6f-8797-eab0b8066b1f" />



8. Vizualizácie a analytické výstupy
Projekt obsahuje viacero analytických pohľadov pripravených na vizualizáciu.
Každá vizualizácia obsahuje:
•	SQL dotaz
•	popis
•	business interpretáciu

VIZ #1 – Trend CPC v čase

<img width="748" height="292" alt="image" src="https://github.com/user-attachments/assets/65949776-1b24-4d94-8b21-980feb09765b" />
<img width="945" height="453" alt="image" src="https://github.com/user-attachments/assets/2946c049-e269-411d-a03f-66edc7ec3c4e" />


VIZ #2 – Top kampane podľa spendu

<img width="639" height="343" alt="image" src="https://github.com/user-attachments/assets/3535a128-8e7e-4307-9927-e67e041671ed" />
<img width="945" height="551" alt="image" src="https://github.com/user-attachments/assets/22d34b27-2e42-4fef-bb47-5330854929a7" />


VIZ #3: Efektívnosť kampaní (Spend vs. Reach) - SCATTER PLOT

<img width="748" height="336" alt="image" src="https://github.com/user-attachments/assets/aa4602ac-0578-482d-a5a1-8819126fc60e" />
<img width="945" height="539" alt="image" src="https://github.com/user-attachments/assets/4091b758-7053-4e77-8d88-8413a78d0f71" />


VIZ #4: CTR podľa cieľov kampánie - BAR CHART

<img width="683" height="352" alt="image" src="https://github.com/user-attachments/assets/7a2a95a8-63eb-4d46-84ad-6d84383223f9" />
<img width="945" height="550" alt="image" src="https://github.com/user-attachments/assets/a9496d71-3da4-4119-9370-9009d7b175cc" />
 


VIZ #5: Top annonce podľa ROI (Spend vs. Engagement) – TABLE

<img width="736" height="488" alt="image" src="https://github.com/user-attachments/assets/eea4869e-1209-4cc6-b1e5-d859bc09d1f5" />
<img width="945" height="557" alt="image" src="https://github.com/user-attachments/assets/e8c52809-4eac-46db-93af-65025fdfff28" />


VIZ #6: Sezónnosť - Výdavky podľa dňa týždňa – HEATMAP

<img width="533" height="473" alt="image" src="https://github.com/user-attachments/assets/41020e1a-d8d2-439c-a85a-471cf00eafb9" />
<img width="945" height="524" alt="image" src="https://github.com/user-attachments/assets/5da17558-71d6-4ddb-8bf1-e7223763a934" />

 
VIZ #7: Kumulatívne výdavky podľa kampánie (WINDOW FUNCTION) - AREA CHART

<img width="909" height="295" alt="image" src="https://github.com/user-attachments/assets/94373c2f-25d5-4266-b404-d6cf8dcce61a" />
<img width="945" height="433" alt="image" src="https://github.com/user-attachments/assets/d97bf3ff-a1b6-45ce-a10b-dc091ac6ae5b" />


VIZ #8: Porovnanie CPC v čase s LAG window function - LINE CHART

<img width="722" height="373" alt="image" src="https://github.com/user-attachments/assets/fffc2297-56df-4e94-add2-bf021a7ac7b9" />
<img width="945" height="456" alt="image" src="https://github.com/user-attachments/assets/b6280688-018d-49b9-9467-221ec1fc8386" />


VIZ #9: Ranking annoncí v 7-dňovom okne (RANK window function) - DETAILED TABLE

<img width="587" height="390" alt="image" src="https://github.com/user-attachments/assets/d559c289-21fb-4379-8269-475f4d9f812b" />
<img width="945" height="430" alt="image" src="https://github.com/user-attachments/assets/34eb0dee-1321-47be-ba22-fe55d1d74c05" />
<img width="945" height="441" alt="image" src="https://github.com/user-attachments/assets/bf271d7b-0be8-4cc5-8778-55b5c02fda37" />


VIZ #10: Performance Dashboard - Overview Metrics - KPI CARD

<img width="754" height="453" alt="image" src="https://github.com/user-attachments/assets/77a74d4b-ae8a-44f9-a7cb-333ae013fd2c" />
<img width="945" height="187" alt="image" src="https://github.com/user-attachments/assets/301dd7bd-21a1-49a3-b92d-7bfb84a5550b" />

9. Performance Dashboard
Dashboard poskytuje high-level prehľad pre manažment:
•	celkové výdavky
•	impressions, clicks, reach
•	priemerné CPC, CTR, CPM
•	trvanie kampaní

10. Kľúčové insights
•	Star Schema výrazne zjednodušuje analytické dotazy
•	Window functions umožňujú pokročilé analýzy bez externých nástrojov
•	Dáta sú pripravené na multi-platform rozšírenie

11. Technické nástroje
•	Snowflake
•	Snowflake Marketplace
•	SQL
•	Vizualizačný nástroj (Power BI / Tableau / Looker)

12. Možné rozšírenia projektu
•	Incrementálny load
•	Alerting na anomálie
•	Prediktívne modelovanie
•	Integrácia ďalších reklamných platforiem

Autori
•	Alexander Krobot
•	Filip Samko
Dátum: Január 2026
Verzia: 1.0

Referencie
•	Snowflake Documentation
•	Ralph Kimball – The Data Warehouse Toolkit
•	Window Functions in SQL
