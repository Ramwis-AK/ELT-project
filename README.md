ELT proces Facebook Ads Analytics v Snowflake
Popis projektu
Tento projekt sa zameriava na návrh a implementáciu komplexného ELT procesu (Extract–Load–Transform) v prostredí Snowflake, vrátane vytvorenia dimenzionálneho dátového modelu typu Star Schema.
Cieľom je analyzovať výkonnosť Facebook Ads kampaní na základe denných metrík a poskytnúť dátový základ pre analytické a reportingové použitie.
Projekt simuluje reálny analytický scenár z marketingového prostredia, kde je kladený dôraz na škálovateľnosť riešenia, čistotu dát a znovupoužiteľnosť dátového modelu.
________________________________________
1. Úvod a popis zdrojových dát
1.1 Tematické zameranie a zdôvodnenie výberu
Projekt je zameraný na analýzu digitálnych reklamných kampaní Facebook Ads. Výber datasetu bol motivovaný najmä jeho praktickým využitím v biznis prostredí a vysokou relevanciou pre marketingové rozhodovanie.
Hlavné dôvody výberu:
•	Biznisová relevancia – umožňuje analyzovať výkonnosť kampaní, optimalizovať rozpočty a porovnávať reklamné stratégie
•	Praktická aplikovateľnosť – výstupy sú využiteľné pri optimalizácii CPC, CPM, cieľov kampaní a targetingu
•	Škálovateľnosť riešenia – rovnaký ELT prístup je aplikovateľný aj na iné reklamné platformy (Google Ads, TikTok Ads, LinkedIn Ads)
•	Dostupnosť dát – dataset je dostupný prostredníctvom Snowflake Marketplace
________________________________________
1.2 Zdrojové dáta a ich štruktúra
Zdroj dát:
Snowflake Marketplace – AD_DATA_FUSION.FACEBOOK_ADS
<img width="891" height="327" alt="image" src="https://github.com/user-attachments/assets/e1f138e5-4b22-413f-a594-c070d3a655a8" />
<img width="849" height="636" alt="image" src="https://github.com/user-attachments/assets/d9eec184-1d06-4f9a-9526-ed5ac5a3105d" />

Dataset obsahuje historické denné dáta o výkonnosti reklám a súvisiace metadata. Základné tabuľky:
Tabuľka	Popis
ADS_INSIGHTS	Denné metriky výkonnosti reklám
CAMPAIGNS	Informácie o kampaniach a ich cieľoch
ADSETS	Skupiny reklám s definovaným targetingom
ADS	Jednotlivé reklamné kreatívy
ACCOUNTS	Facebook Business účty
________________________________________
1.3 Kľúčové metriky
Hlavné analyzované metriky zahŕňajú:
•	IMPRESSIONS – počet zobrazení reklamy
•	CLICKS – počet kliknutí
•	SPEND – suma vynaložená na reklamu (USD)
•	REACH – počet unikátnych používateľov
•	FREQUENCY – priemerný počet zobrazení na používateľa
•	CTR (Click-Through Rate) – pomer kliknutí k impressions
•	CPM (Cost per Mille) – cena za 1000 zobrazení
•	CPC (Cost per Click) – cena za jedno kliknutie
________________________________________
1.4 Ciele analýzy
Hlavnými cieľmi projektu sú:
1.	Identifikácia najvýkonnejších kampaní, adsetov a reklám
2.	Analýza nákladovosti kampaní pomocou metrík CPC a CPM
3.	Sledovanie trendov výkonnosti v čase
4.	Porovnanie výkonu kampaní podľa ich cieľov
5.	Vytvorenie dátového modelu vhodného pre reporting a BI nástroje
________________________________________
1.5 ERD diagram pôvodnej dátovej štruktúry
Pôvodná štruktúra datasetu je založená na normalizovanom modeli, kde sú jednotlivé entity (účty, kampane, adsety, reklamy) navzájom prepojené prostredníctvom identifikátorov.

________________________________________
2. Návrh dimenzionálneho modelu (Star Schema)
Na analytické účely bol navrhnutý dimenzionálny model typu Star Schema, ktorý umožňuje efektívne agregácie a jednoduché použitie v BI nástrojoch.
