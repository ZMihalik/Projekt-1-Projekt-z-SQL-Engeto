# Průvodní listina k projektu

## Dostupnost základních potravin – mzdy vs. ceny potravin v ČR

---

## Obsah

1. [Úvod a cíl projektu](#1-úvod-a-cíl-projektu)
2. [Použité datové sady](#2-použité-datové-sady)
3. [Příprava: pohledy (views)](#3-příprava-pohledy-views)
4. [Výzkumné otázky a odpovědi](#4-výzkumné-otázky-a-odpovědi)
   - [Otázka 1 – vývoj mezd v odvětvích](#otázka-1--vývoj-mezd-v-odvětvích)
   - [Otázka 2 – kolik mléka a chleba si lze koupit](#otázka-2--kolik-mléka-a-chleba-si-lze-koupit)
   - [Otázka 3 – nejpomaleji zdražující potravina](#otázka-3--nejpomaleji-zdražující-potravina)
   - [Otázka 4 – rok s výrazným zdražením potravin](#otázka-4--rok-s-výrazným-zdražením-potravin)
   - [Otázka 5 – vliv HDP na mzdy a ceny](#otázka-5--vliv-hdp-na-mzdy-a-ceny)
5. [Finální výstupní tabulky](#5-finální-výstupní-tabulky)

---

## 1. Úvod a cíl projektu

Cílem projektu je zodpovědět pět výzkumných otázek, které se zabývají dostupností základních potravin pro širokou veřejnost.

Analytické oddělení nezávislé společnosti, které se zabývá životní úrovní občanů, poskytne informace tiskovému oddělení. To bude výsledky prezentovat na konferenci zaměřené na tuto oblast. Datové podklady umožní porovnat dostupnost potravin na základě průměrných příjmů za určité časové období.

### Výzkumné otázky

| # | Otázka |
|---|--------|
| 1 | Rostou v průběhu let mzdy ve všech odvětvích, nebo v některých klesají? |
| 2 | Kolik je možné si koupit litrů mléka a kilogramů chleba za první a poslední srovnatelné období v dostupných datech cen a mezd? |
| 3 | Která kategorie potravin zdražuje nejpomaleji (je u ní nejnižší percentuální meziroční nárůst)? |
| 4 | Existuje rok, ve kterém byl meziroční nárůst cen potravin výrazně vyšší než růst mezd (větší než 10 %)? |
| 5 | Má výška HDP vliv na změny ve mzdách a cenách potravin? Neboli, pokud HDP vzroste výrazněji v jednom roce, projeví se to na cenách potravin či mzdách ve stejném nebo následujícím roce výraznějším růstem? |

Součástí výstupu je také tabulka, ve které jsou vidět dodatečná data o dalších evropských státech.

---

## 2. Použité datové sady

| Tabulka | Obsah |
|---------|-------|
| `czechia_payroll` | informace o mzdách v různých odvětvích za několikaleté období |
| `czechia_payroll_industry_branch` | číselník odvětví v tabulce mezd |
| `czechia_price` | informace o cenách vybraných potravin za několikaleté období |
| `czechia_price_category` | číselník kategorií potravin |
| `countries` | informace o zemích světa |
| `economies` | HDP, GINI, daňová zátěž a další ukazatele pro daný stát a rok |

---

## 3. Příprava: pohledy (views)

Pohledy (views), na kterých je celá analýza postavena.

### 3.1 Průměrná roční mzda podle odvětví

Zohledňuji:

- `value_type_code = 5958` → „Průměrná hrubá mzda na zaměstnance“
- `calculation_code = 100` → „fyzický“ (drží jednu metodiku konzistentně)

Vytvořila jsem aliasy tabulek, aby se nemusely používat dlouhé názvy:

| Alias | Význam |
|-------|--------|
| `p` | payroll |
| `ib` | industry branch |

Ve vzniklé tabulce vidím průměrnou hrubou mzdu na zaměstnance v jednotlivých letech. Vidím také názvy odvětví, nejen kód odvětví.

```sql
CREATE VIEW v_wages_by_industry_yearly AS
SELECT
    p.industry_branch_code,
    ib.name AS industry_name,
    p.payroll_year AS year,
    AVG(p.value) AS avg_wage
FROM czechia_payroll p
JOIN czechia_payroll_industry_branch ib
    ON ib.code = p.industry_branch_code
WHERE p.value_type_code  = 5958
  AND p.calculation_code = 100
GROUP BY p.industry_branch_code, ib.name, p.payroll_year;
```

### 3.2 Průměrná roční mzda za celou ČR (napříč odvětvími)

Tyto informace budu potřebovat pro otázky 2, 4 a 5.

```sql
CREATE VIEW v_wages_total_yearly AS
SELECT
    payroll_year AS year,
    AVG(value)   AS avg_wage
FROM czechia_payroll
WHERE value_type_code  = 5958
  AND calculation_code = 100
GROUP BY payroll_year;
```

### 3.3 Průměrná roční cena za každou kategorii potravin

| Alias | Význam |
|-------|--------|
| `pr` | price |
| `pc` | price category |

Ve výsledné tabulce vidím průměrnou roční cenu za každou kategorii potravin.

```sql
CREATE VIEW v_price_by_category_yearly AS
SELECT
    pr.category_code,
    pc.name AS category_name,
    pc.price_unit,
    EXTRACT(YEAR FROM pr.date_from)::INT AS year,
    AVG(pr.value) AS avg_price
FROM czechia_price pr
JOIN czechia_price_category pc
    ON pc.code = pr.category_code
GROUP BY pr.category_code, pc.name, pc.price_unit, EXTRACT(YEAR FROM pr.date_from);
```

### 3.4 Roky, ve kterých existují data současně pro mzdy i ceny (srovnatelné období)

```sql
CREATE VIEW v_common_years AS
SELECT year FROM v_wages_total_yearly
INTERSECT
SELECT year FROM v_price_by_category_yearly;
```

---

## 4. Výzkumné otázky a odpovědi

## Otázka 1 – vývoj mezd v odvětvích

> **Rostou v průběhu let mzdy ve všech odvětvích, nebo v některých klesají?**

### Meziroční srovnání (růst / pokles) po jednotlivých letech a odvětvích

`yoy` = *year over year* (meziroční)

```sql
WITH yoy AS (
    SELECT
        industry_branch_code,
        industry_name,
        year,
        avg_wage,
        LAG(avg_wage) OVER (PARTITION BY industry_branch_code ORDER BY year) AS prev_wage
    FROM v_wages_by_industry_yearly
)
SELECT
    industry_name,
    year,
    ROUND(avg_wage::numeric, 0) AS avg_wage,
    ROUND(prev_wage::numeric, 0) AS prev_wage,
    CASE
        WHEN prev_wage IS NULL       THEN 'první rok'
        WHEN avg_wage > prev_wage    THEN 'růst'
        WHEN avg_wage < prev_wage    THEN 'pokles'
        ELSE 'beze změny'
    END AS trend
FROM yoy
ORDER BY industry_name, year;
```

### Souhrn za celé období (první vs. poslední rok)

Jednoznačná odpověď na otázku: ve všech odvětvích mzda rostla.

| Zkratka | Význam |
|---------|--------|
| `w1` | wage first year (mzda v prvním roce) |
| `w2` | wage last year (mzda v posledním roce) |
| `bounds` | vymezení nejstaršího a nejnovějšího roku |
| `pct` | percent (změna v procentech) |

```sql
WITH bounds AS (
    SELECT industry_branch_code, MIN(year) AS first_year, MAX(year) AS last_year
    FROM v_wages_by_industry_yearly
    GROUP BY industry_branch_code
)
SELECT
    w1.industry_name,
    ROUND(w1.avg_wage::numeric, 0) AS wage_first_year,
    ROUND(w2.avg_wage::numeric, 0) AS wage_last_year,
    ROUND((100.0 * (w2.avg_wage - w1.avg_wage) / w1.avg_wage)::numeric, 1) AS change_pct,
    CASE WHEN w2.avg_wage < w1.avg_wage THEN 'POKLES' ELSE 'RŮST' END AS overall_trend
FROM bounds b
JOIN v_wages_by_industry_yearly w1 ON w1.industry_branch_code = b.industry_branch_code AND w1.year = b.first_year
JOIN v_wages_by_industry_yearly w2 ON w2.industry_branch_code = b.industry_branch_code AND w2.year = b.last_year
ORDER BY change_pct ASC;
```

### Odpověď

> Dlouhodobě mzdy vzrostly v každém odvětví, ale ne každý rok. Poklesy byly následující:

#### Přehled podle odvětví

| Odvětví | Roky poklesu |
|---------|--------------|
| Činnosti v oblasti nemovitostí | 2013, 2020 |
| Informační a komunikační činnosti | 2013 |
| Kulturní, zábavní a rekreační činnosti | 2011, 2013, 2021 |
| Peněžnictví a pojišťovnictví | 2013 |
| Profesní, vědecké a technické činnosti | 2010, 2013 |
| Stavebnictví | 2013, 2021 |
| Těžba a dobývání | 2009, 2013, 2014, 2016 |
| Ubytování, stravování a pohostinství | 2009, 2011, 2020 |
| Velkoobchod a maloobchod; opravy a údržba motorových vozidel | 2009, 2013 |
| Veřejná správa a obrana; povinné sociální zabezpečení | 2010, 2011, 2021 |
| Výroba a rozvod elektřiny, plynu, tepla a klimatiz. vzduchu | 2013, 2015 |
| Vzdělávání | 2010, 2021 |
| Zásobování vodou; činnosti související s odpady a sanacemi | 2013 |
| Zemědělství, lesnictví, rybářství | 2009, 2021 |
| Zpracovatelský průmysl | 2020 |

#### Přehled podle roků a pravděpodobné příčiny

| Rok | Odvětví s poklesem mezd | Pravděpodobná příčina |
|-----|-------------------------|-----------------------|
| **2009** | Těžba a dobývání<br>Ubytování, stravování a pohostinství<br>Velkoobchod a maloobchod; opravy a údržba motorových vozidel<br>Zemědělství, lesnictví, rybářství | Světová finanční a hospodářská krize. Poklesla poptávka, firmy omezily prémie a přesčasy. Nejvíce to pocítila těžba a odvětví závislá na spotřebě. |
| **2010** | Profesní, vědecké a technické činnosti<br>Veřejná správa a obrana; povinné sociální zabezpečení<br>Vzdělávání | Úsporná opatření vlády ve veřejném sektoru (snižování platů a odměn). |
| **2011** | Kulturní, zábavní a rekreační činnosti<br>Ubytování, stravování a pohostinství<br>Veřejná správa a obrana; povinné sociální zabezpečení | Pokračující škrty ve veřejném sektoru. U pohostinství a kultury slabá domácí spotřeba a vyšší DPH. |
| **2013** | Činnosti v oblasti nemovitostí<br>Informační a komunikační činnosti<br>Kulturní, zábavní a rekreační činnosti<br>Peněžnictví a pojišťovnictví<br>Profesní, vědecké a technické činnosti<br>Stavebnictví<br>Těžba a dobývání<br>Velkoobchod a maloobchod; opravy a údržba motorových vozidel<br>Výroba a rozvod elektřiny, plynu, tepla a klimatiz. vzduchu<br>Zásobování vodou; činnosti související s odpady a sanacemi | Recese 2012–2013. Navíc firmy pravděpodobně vyplatily část odměn už koncem roku 2012, před zavedením vyšší daně pro vysoké příjmy od roku 2013. Tomu odpovídá nejhlubší pokles ve finančnictví, kde jsou vysoké příjmy a prémie. Energetika mohla být zasažena i poklesem cen elektřiny. |
| **2014** | Těžba a dobývání | Útlum těžby a nižší poptávka po uhlí. |
| **2015** | Výroba a rozvod elektřiny, plynu, tepla a klimatiz. vzduchu | Pravděpodobně pokles cen elektřiny a restrukturalizace v oboru. |
| **2016** | Těžba a dobývání | Pokračující útlum těžby. |
| **2020** | Činnosti v oblasti nemovitostí<br>Ubytování, stravování a pohostinství<br>Zpracovatelský průmysl | Pandemie covidu. Lockdowny zasáhly především služby. U nemovitostí se mohla propadnout poptávka a provize. |
| **2021** | Kulturní, zábavní a rekreační činnosti<br>Stavebnictví<br>Veřejná správa a obrana; povinné sociální zabezpečení<br>Vzdělávání<br>Zemědělství, lesnictví, rybářství | Dozvuky covidu. Kultura byla dlouho zavřená a slábla státní podpora. Ve veřejném sektoru došlo ke zmrazení platů po růstu v roce 2020. U zemědělství pravděpodobně výkyv prémií a sezónní vlivy. |

---

## Otázka 2 – kolik mléka a chleba si lze koupit

> **Kolik je možné si koupit litrů mléka a kilogramů chleba za první a poslední srovnatelné období v dostupných datech cen a mezd?**

### Mezikrok: ceny mléka a chleba jen v letech, ve kterých máme i mzdu

```sql
WITH milk_bread AS (
    SELECT category_name, price_unit, year, avg_price
    FROM v_price_by_category_yearly
    WHERE category_name IN ('Mléko polotučné pasterované', 'Chléb konzumní kmínový')
      AND year IN (SELECT year FROM v_common_years)
),
bounds AS (SELECT MIN(year) AS first_year, MAX(year) AS last_year FROM v_common_years)
SELECT
    mb1.category_name,
    mb1.price_unit,
    b.first_year,
    ROUND((w1.avg_wage / mb1.avg_price)::numeric, 1)  AS mnozstvi_prvni_rok,
    b.last_year,
    ROUND((w2.avg_wage / mb2.avg_price)::numeric, 1)  AS mnozstvi_posledni_rok
FROM bounds b
JOIN milk_bread mb1 ON mb1.year = b.first_year
JOIN milk_bread mb2 ON mb2.year = b.last_year AND mb2.category_name = mb1.category_name
JOIN v_wages_total_yearly w1 ON w1.year = b.first_year
JOIN v_wages_total_yearly w2 ON w2.year = b.last_year;
```

### Odpověď

| Rok | Mléko (litry) | Chléb (kilogramy) |
|-----|--------------:|------------------:|
| 2006 | 1 404 | 1 257,2 |
| 2018 | 1 611,3 | 1 317,4 |

> V roce 2006 si bylo možné koupit 1 404 litrů mléka a 1 257,2 kilogramu chleba, v roce 2018 už 1 611,3 litru mléka a 1 317,4 kilogramu chleba.

---

## Otázka 3 – nejpomaleji zdražující potravina

> **Která kategorie potravin zdražuje nejpomaleji (nejnižší průměrný meziroční procentuální nárůst)?**

**Počet meziročních srovnání** je počet dvojic sousedních let, ze kterých se počítá průměr. Například 12 srovnání znamená, že kategorie má data za 13 let. Jakostní víno bílé má 3 srovnání, což znamená, že pro něj máme data pouze za 4 roky.

`yoy` = *year over year* (meziroční)

```sql
WITH yoy AS (
    SELECT
        category_code,
        category_name,
        year,
        avg_price,
        LAG(avg_price) OVER (PARTITION BY category_code ORDER BY year) AS prev_price
    FROM v_price_by_category_yearly
),
yoy_pct AS (
    SELECT
        category_code,
        category_name,
        year,
        100.0 * (avg_price - prev_price) / prev_price AS pct_change
    FROM yoy
    WHERE prev_price IS NOT NULL AND prev_price > 0
)
SELECT
    category_name,
    ROUND(AVG(pct_change)::numeric, 2) AS avg_yoy_growth_pct,
    COUNT(*)                            AS pocet_meziročních_srovnání
FROM yoy_pct
GROUP BY category_name
ORDER BY avg_yoy_growth_pct ASC;
```

### Odpověď

Nejnižší hodnota ve sloupci `avg_yoy_growth_pct` označuje potravinu, která zdražuje nejpomaleji. Záporné hodnoty však lze interpretovat tak, že potravina nejenže nezdražovala, ale dokonce zlevňovala. Nejpomaleji, v pravém slova smyslu, tedy zdražovaly **banány žluté**.

| Pořadí | Kategorie | Průměrný meziroční růst |
|-------:|-----------|------------------------:|
| 1 | Cukr krystalový | −1,92 % (zlevňuje) |
| 2 | Rajská jablka červená kulatá | −0,74 % (zlevňují) |
| 3 | **Banány žluté** | **+0,81 %** (nejpomaleji zdražující) |

---

## Otázka 4 – rok s výrazným zdražením potravin

> **Existuje rok, ve kterém byl meziroční nárůst cen potravin výrazně vyšší než růst mezd (rozdíl > 10 p. b.)?**

Ceny různých kategorií mají různé jednotky (kg/l/ks), proto nelze zprůměrovat `avg_price` napříč kategoriemi. Každou kategorii nejdříve převedu na cenový index (první rok = 100) a pak indexy zprůměruji. Tím dostanu jeden souhrnný „cenový index ČR“.

```sql
WITH base_price AS (
    SELECT
        category_code,
        FIRST_VALUE(avg_price) OVER (PARTITION BY category_code ORDER BY year) AS base,
        year,
        avg_price
    FROM v_price_by_category_yearly
),
category_index AS (
    SELECT category_code, year, 100.0 * avg_price / base AS price_index
    FROM base_price
),
overall_price_index AS (
    SELECT year, AVG(price_index) AS avg_price_index
    FROM category_index
    GROUP BY year
),
price_index_yoy AS (
    SELECT
        year,
        avg_price_index,
        LAG(avg_price_index) OVER (ORDER BY year) AS prev_index
    FROM overall_price_index
),
price_pct AS (
    SELECT year, 100.0 * (avg_price_index - prev_index) / prev_index AS price_growth_pct
    FROM price_index_yoy
    WHERE prev_index IS NOT NULL
),
wage_pct AS (
    SELECT
        year,
        100.0 * (avg_wage - LAG(avg_wage) OVER (ORDER BY year))
              / LAG(avg_wage) OVER (ORDER BY year) AS wage_growth_pct
    FROM v_wages_total_yearly
)
SELECT
    p.year,
    ROUND(p.price_growth_pct::numeric, 2) AS ceny_zmena_pct,
    ROUND(w.wage_growth_pct::numeric, 2)  AS mzdy_zmena_pct,
    ROUND((p.price_growth_pct - w.wage_growth_pct)::numeric, 2) AS rozdil_p_b
FROM price_pct p
JOIN wage_pct w ON w.year = p.year
WHERE (p.price_growth_pct - w.wage_growth_pct) > 10
ORDER BY p.year;
```

### Odpověď

Po spuštění příkazu nebyla vrácena žádná data. To znamená, že neexistuje rok, ve kterém by meziroční nárůst cen potravin byl výrazně vyšší než růst mezd (rozdíl > 10 p. b.).

---

## Otázka 5 – vliv HDP na mzdy a ceny

> **Má výška HDP vliv na mzdy a ceny potravin? Projeví se to ve stejném roce, nebo až v roce následujícím?**

`gdp` = *gross domestic product* (hrubý domácí produkt, HDP)

### Krok 1: Ověření názvu země

Výsledek: `Czech Republic`

```sql
SELECT DISTINCT country
FROM economies
WHERE country ILIKE '%czech%';
```

### Krok 2: O kolik procent rostlo HDP v každém roce?

```sql
CREATE VIEW v_gdp_growth AS
SELECT
    year,
    100.0 * (gdp - LAG(gdp) OVER (ORDER BY year))
          / LAG(gdp) OVER (ORDER BY year) AS gdp_growth_pct
FROM economies
WHERE country = 'Czech Republic';
```

### Krok 3: O kolik procent rostly mzdy v každém roce?

```sql
create VIEW v_wage_growth AS
SELECT
    year,
    100.0 * (avg_wage - LAG(avg_wage) OVER (ORDER BY year))
          / LAG(avg_wage) OVER (ORDER BY year) AS wage_growth_pct
FROM v_wages_total_yearly;
```

### Krok 4: O kolik procent rostly ceny potravin v každém roce?

Nejprve zjistím změnu ceny pro každou kategorii zvlášť, poté průměr přes všechny kategorie.

```sql
create VIEW v_price_growth AS
SELECT
    year,
    AVG(pct_change) AS price_growth_pct
FROM (
    SELECT
        category_code,
        year,
        100.0 * (avg_price - LAG(avg_price) OVER (PARTITION BY category_code ORDER BY year))
              / LAG(avg_price) OVER (PARTITION BY category_code ORDER BY year) AS pct_change
    FROM v_price_by_category_yearly
) t
WHERE pct_change IS NOT NULL
GROUP BY year;
```

### Krok 5: Spojení pohledů (views) do jedné tabulky

```sql
SELECT
    g.year,
    ROUND(g.gdp_growth_pct::numeric, 2)    AS hdp_zmena_pct,
    ROUND(w.wage_growth_pct::numeric, 2)   AS mzdy_stejny_rok,
    ROUND(w2.wage_growth_pct::numeric, 2)  AS mzdy_nasledujici_rok,
    ROUND(p.price_growth_pct::numeric, 2)  AS ceny_stejny_rok,
    ROUND(p2.price_growth_pct::numeric, 2) AS ceny_nasledujici_rok
FROM v_gdp_growth g
JOIN v_wage_growth w   ON w.year  = g.year
JOIN v_price_growth p  ON p.year  = g.year
LEFT JOIN v_wage_growth w2  ON w2.year = g.year + 1
LEFT JOIN v_price_growth p2 ON p2.year = g.year + 1
WHERE g.gdp_growth_pct IS NOT NULL
ORDER BY g.year;
```

### Výsledná tabulka

Hodnoty jsou v procentech (meziroční změna). U roku 2018 chybí hodnota `ceny_nasledujici_rok`, protože nemáme data o cenách za rok 2019.

| Rok | HDP | Mzdy – stejný rok | Mzdy – následující rok | Ceny – stejný rok | Ceny – následující rok |
|----:|----:|------------------:|-----------------------:|------------------:|-----------------------:|
| 2007 | 5,57 | 6,82 | 8,05 | 9,26 | 8,92 |
| 2008 | 2,69 | 8,05 | 3,25 | 8,92 | −6,58 |
| 2009 | −4,66 | 3,25 | 2,00 | −6,58 | 1,51 |
| 2010 | 2,43 | 2,00 | 2,27 | 1,51 | 4,84 |
| 2011 | 1,76 | 2,27 | 3,11 | 4,84 | 7,47 |
| 2012 | −0,79 | 3,11 | −1,49 | 7,47 | 6,01 |
| 2013 | −0,05 | −1,49 | 2,56 | 6,01 | −0,63 |
| 2014 | 2,26 | 2,56 | 2,45 | −0,63 | −0,69 |
| 2015 | 5,39 | 2,45 | 3,70 | −0,69 | −1,41 |
| 2016 | 2,54 | 3,70 | 6,43 | −1,41 | 7,06 |
| 2017 | 5,17 | 6,43 | 7,57 | 7,06 | 2,41 |
| 2018 | 3,20 | 7,57 | 7,05 | 2,41 | – |

### Vyhodnocení

Následně porovnám hodnoty ve sloupci `hdp_zmena_pct` s hodnotami ve sloupcích `mzdy_stejny_rok` a `mzdy_nasledujici_rok` a poté `hdp_zmena_pct` s hodnotami ve sloupcích `ceny_stejny_rok` a `ceny_nasledujici_rok`. Sleduji, co se děje se mzdami a cenami, když HDP klesá nebo roste.

**Mzdy:** Vývoj HDP se ve mzdách projevuje spíše s ročním zpožděním.

- Po propadu HDP v roce 2009 (−4,66 %) zpomalil růst mezd z 8,05 % v roce 2008 na 3,25 % v roce 2009 a na 2,00 % v roce 2010.
- Po poklesu HDP v roce 2012 (−0,79 %) mzdy v následujícím roce 2013 dokonce klesly (−1,49 %).
- Po silném růstu HDP v roce 2015 (5,39 %) se růst mezd zrychlil až v dalších letech: na 3,70 % v roce 2016, 6,43 % v roce 2017 a 7,57 % v roce 2018.
- V roce 2007 (HDP +5,57 %) mzdy rostly rychle už ve stejném roce (6,82 %) a ještě rychleji v roce následujícím (8,05 %).

**Ceny potravin:** Vztah k HDP je méně jednoznačný.

- Ve stejném roce reagovaly ceny na propad HDP v roce 2009 (−6,58 %) a na silný růst HDP v letech 2007 a 2017 (+9,26 % a +7,06 %).
- Naopak po silném růstu HDP v roce 2015 ceny nerostly (−0,69 % v roce 2015 a −1,41 % v roce 2016).
- V letech 2012 a 2013, kdy HDP mírně kleslo, ceny potravin vzrostly o 7,47 % a 6,01 %.

**Shrnutí:** Změna HDP se na mzdách zřetelněji projevila v následujícím roce, a to zejména v letech 2009–2010, 2013 a 2015–2018. U cen potravin se souvislost s HDP projevila pouze v některých letech (2007, 2009 a 2017) a ve zbývajících letech se neprojevila.

Data naznačují souvislost mezi HDP a mzdami a cenami, ale zcela ji nedokazují, a proto nelze s jistotou tvrdit, že pohyb HDP ovlivňuje pohyb mezd a cen.

---

## 5. Finální výstupní tabulky

### 5.1 Primární tabulka

Tabulka obsahuje mzdy a ceny potravin v ČR za společné roky (`v_common_years`).

| Sloupec | Popis |
|---------|-------|
| `rok` | rok |
| `kategorie` | typ ukazatele: `mzda` nebo `cena` |
| `odvetvi_produkt` | název odvětví (mzdy) nebo kategorie potraviny (ceny) |
| `hodnota` | průměrná roční mzda nebo cena |
| `jednotka` | jednotka hodnoty (Kč, kg, l apod.) |

```sql
CREATE TABLE t_zuzana_mihalikova_project_sql_primary_final AS
SELECT
    year                          AS rok,
    'mzda'                        AS kategorie,
    industry_name                 AS odvetvi_produkt,
    ROUND(avg_wage::numeric, 0)   AS hodnota,
    'Kč'                          AS jednotka
FROM v_wages_by_industry_yearly
WHERE year IN (SELECT year FROM v_common_years);
```

```sql
INSERT INTO t_zuzana_mihalikova_project_sql_primary_final
SELECT
    year                          AS rok,
    'cena'                        AS kategorie,
    category_name                 AS odvetvi_produkt,
    ROUND(avg_price::numeric, 2)  AS hodnota,
    price_unit                    AS jednotka
FROM v_price_by_category_yearly
WHERE year IN (SELECT year FROM v_common_years);
```

### 5.2 Sekundární tabulka

Tabulka obsahuje HDP, GINI koeficient a populaci dalších evropských států za stejné roky jako primární tabulka.

| Sloupec | Popis |
|---------|-------|
| `country` | stát |
| `rok` | rok |
| `gdp` | hrubý domácí produkt |
| `gini` | GINI koeficient |
| `population` | počet obyvatel |

```sql
CREATE TABLE t_zuzana_mihalikova_project_sql_secondary_final AS
SELECT
    c.country,
    e.year        AS rok,
    e.gdp,
    e.gini,
    e.population
FROM economies e
JOIN countries c
    ON c.country = e.country
WHERE c.continent = 'Europe'
  AND c.country <> 'Czech Republic'
  AND e.year IN (SELECT year FROM v_common_years);
```

### 5.3 Chybějící hodnoty

```sql
SELECT 'primary_final - chybějící hodnota' AS kontrola, COUNT(*) AS pocet
FROM t_zuzana_mihalikova_project_SQL_primary_final WHERE hodnota IS NULL
UNION ALL
SELECT 'secondary_final - chybějící gdp', COUNT(*)
FROM t_zuzana_mihalikova_project_SQL_secondary_final WHERE gdp IS NULL
UNION ALL
SELECT 'secondary_final - chybějící gini', COUNT(*)
FROM t_zuzana_mihalikova_project_SQL_secondary_final WHERE gini IS NULL;
```

**Výsledek:**

| Kontrola | Počet chybějících hodnot |
|----------|-------------------------:|
| primary_final – chybějící hodnota | 0 |
| secondary_final – chybějící gdp | 37 |
| secondary_final – chybějící gini | 124 |
