# Case study: Global HIV Treatment Effectiveness Analysis

Extended technical write-up. The main [README](../README.md) covers the
headline story — this document covers how it was built.

## 1. Business objective

WHO (as the framing client for this analysis) needed a single, decision-ready
view answering three questions:

1. Is HIV treatment coverage actually reducing mortality, or just extending
   the epidemic's timeline?
2. How does the relationship between infections, deaths, and treatment access
   vary by country?
3. Which specific countries most urgently need additional treatment
   investment?

## 2. Data sources

- WHO official published HIV statistics, 2000–2024.
- 150 countries, 4 indicators: new infections, deaths, people living with HIV
  (PLHIV), and ART (antiretroviral therapy) coverage.
- 9,763 total records after cleaning.

## 3. Data modelling — why a star schema

The raw data arrived as a long, indicator-per-row table (country, year,
indicator, value). Rather than building measures directly against that flat
structure, the data was split into a star schema:

- **FactHIV** — the numeric core: `CountryKey`, `IndicatorKey`, `YearKey`,
  `Value`.
- **DimCountry** — `country`, `continent`, `location_code`.
- **DimIndicator** — `indicator`, `IndicatorType`, `Unit`.
- **DimDate** — `Year`, `Decade`, `EraLabel`, `DataAvailability`.

This matters practically, not just architecturally: separating dimensions
from the fact table means a measure like "average ART coverage" stays
correct no matter which combination of year, country, or region filters is
applied on the report page — and a fifth indicator could be added later by
extending `DimIndicator` without touching the fact table or rebuilding any
visuals.

## 4. Key DAX measures

Ten measures were built in total. The two most load-bearing, in plain terms:

**Mortality rate** — total deaths divided by total people living with HIV,
for the selected filter context:
```dax
Mortality Rate =
DIVIDE(
    CALCULATE(SUM(FactHIV[Value]), DimIndicator[indicator] = "Deaths"),
    CALCULATE(SUM(FactHIV[Value]), DimIndicator[indicator] = "PLHIV")
)
```

**Average ART coverage** — mean treatment coverage across the selected
countries/years:
```dax
Avg ART Coverage =
CALCULATE(
    AVERAGE(FactHIV[Value]),
    DimIndicator[indicator] = "ART Coverage"
)
```

The remaining measures cover year-on-year change, deaths per 1,000 new
infections (used to rank countries independent of population size), and the
high-risk flag used on the priority countries page (new infections > 10,000
**and** ART coverage < 50%, evaluated simultaneously).

## 5. Report structure

- **Page 1 — Global overview.** KPI cards (total PLHIV, deaths, new
  infections, average ART coverage, mortality rate), a 2000–2024 trend line,
  and burden by WHO region.
- **Page 2 — Correlation analysis.** A scatter plot of ART coverage against
  mortality rate by country (bubble size = total PLHIV), plus a ranked bar
  chart of deaths per 1,000 new infections — chosen specifically because it
  controls for population size, so small under-resourced countries aren't
  hidden behind large countries' bigger absolute numbers.
- **Page 3 — Priority countries.** A filtered table of the 7 countries
  meeting both the high-infection and low-coverage criteria, with slicers on
  country, region, and year throughout all three pages.

## 6. Limitations

- ART coverage averages are unweighted by population — a country with 1
  million PLHIV and a country with 10,000 currently contribute equally to
  the global average figure.
- Pre-2020 ART and infection data exists only as 5-year snapshots rather
  than annual records, which limits the granularity of trend detection in
  the earlier part of the time series and delayed visibility of the
  2023–2024 uptick in deaths.
- Data is aggregated at country level; sub-national or regional variation
  within a country isn't visible in this model.

## 7. What I'd do differently next time

- Weight the ART coverage average by population to avoid smaller countries
  distorting the global figure.
- Add a population dimension so per-capita metrics could be calculated
  directly rather than approximated.
- Publish the report to Power BI's web service so it can be explored
  interactively rather than viewed only as static screenshots.
