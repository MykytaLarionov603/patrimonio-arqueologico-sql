# patrimonio-arqueologico-sql

# Archaeological Heritage — Municipality of Águeda

Data analysis project on the archaeological heritage of the municipality of Águeda,
using PostgreSQL to process the data and Power BI to visualize it in an
interactive dashboard with charts and a map.

## Data source
Águeda Municipal Council — Open Data Portal
https://dados.gov.pt/organizations/camara-municipal-de-agueda

## Tools used
- **PostgreSQL + pgAdmin** — data import, processing and querying
- **Power BI** — visualization, charts and interactive map

## Project question
**How is the archaeological heritage of the municipality distributed by parish and by chronological period, and where is it located on the map?**

To answer this, three analyses were carried out:
1. Number of sites per parish
2. Number of sites per chronological period
3. Location of each site on a map

## Findings
- 17 archaeological sites identified in the municipality of Águeda
- Espinhel is the parish with the most sites (4 out of 17)
- By chronological period, Roman-era sites and sites with an Undetermined
  chronology are tied as the most common (5 sites each), followed by
  Prehistory (4 sites) and the Medieval period (3 sites)
- The classification by period is approximate: the original `cronologia`
  field is free text (e.g. "Romano (?)", "Alta Idade Media", "Tardo
  romano/medieval") and was simplified using a keyword-based rule, which
  can fail on word variations (e.g. "Romana" instead of "Romano" was not
  recognized under that category)

## Process — step by step

### 1. Importing the CSV into PostgreSQL
Table creation and import of the original CSV file through pgAdmin
(*Import/Export Data*), with the **Escape** field set to `"` instead of
the default `'`, to correctly handle the single quotes present in the
coordinates (e.g. `40º 39' 12.41" N`).

```sql
CREATE TABLE pa_test (
    id          integer,
    gid         integer,
    codigo      integer,
    dominio     text,
    subdominio  text,
    familia     text,
    objecto     text,
    ident_gene  text,
    ident_part  text,
    regula      text,
    cronologia  text,
    obs         text,
    freguesia   text,
    lat         text,
    lon         text,
    id_patri    integer,
    id_orden    text
);
```

### 2. Data exploration
Initial queries to get familiar with the table: total records, existing
parishes, filters by chronology and by site type.

```sql
SELECT count(*) FROM pa_test;

SELECT DISTINCT freguesia FROM pa_test;

SELECT ident_part FROM pa_test WHERE ident_part ILIKE '%mamoa%';
```

### 3. Sites per parish

```sql
SELECT freguesia, count(*) AS total
FROM pa_test
GROUP BY freguesia
ORDER BY total DESC;
```

### 4. Sites per chronological period
The `cronologia` column had many different free-text values, so they were
grouped into categories using `CASE`:

```sql
SELECT
  CASE
    WHEN cronologia ILIKE '%milenio%' THEN 'Pré-história'
    WHEN cronologia ILIKE '%romano%' THEN 'Romano'
    WHEN cronologia ILIKE '%medi%' THEN 'Medieval'
    ELSE 'Indeterminada'
  END AS periodo,
  count(*) AS total
FROM pa_test
GROUP BY periodo
ORDER BY total DESC;
```

### 5. Coordinate conversion
The coordinates were originally in degrees/minutes/seconds format
(e.g. `40º 39' 12.41" N`) and were converted to decimal degrees, the
format required by Power BI to draw the map:

```sql
ALTER TABLE pa_test ADD COLUMN lat_dec numeric, ADD COLUMN lon_dec numeric;

UPDATE pa_test SET
  lat_dec =  (regexp_split_to_array(lat, '[^0-9.]+'))[1]::numeric
           + (regexp_split_to_array(lat, '[^0-9.]+'))[2]::numeric / 60
           + (regexp_split_to_array(lat, '[^0-9.]+'))[3]::numeric / 3600,
  lon_dec = -( (regexp_split_to_array(lon, '[^0-9.]+'))[1]::numeric
           + (regexp_split_to_array(lon, '[^0-9.]+'))[2]::numeric / 60
           + (regexp_split_to_array(lon, '[^0-9.]+'))[3]::numeric / 3600 );
```

### 6. Views created for Power BI
Instead of importing the entire table, views were created containing the
result of each analysis:

```sql
CREATE VIEW v_sitios_por_freguesia AS
SELECT freguesia, count(*) AS total
FROM pa_test
GROUP BY freguesia
ORDER BY total DESC;

CREATE VIEW v_sitios_por_periodo AS
SELECT
  CASE
    WHEN cronologia ILIKE '%milenio%' THEN 'Pré-história'
    WHEN cronologia ILIKE '%romano%' THEN 'Romano'
    WHEN cronologia ILIKE '%medi%' THEN 'Medieval'
    ELSE 'Indeterminada'
  END AS periodo,
  count(*) AS total
FROM pa_test
GROUP BY periodo
ORDER BY total DESC;

CREATE VIEW v_mapa_sitios AS
SELECT ident_part, freguesia, cronologia, lat_dec, lon_dec
FROM pa_test;
```

### 7. Connecting to Power BI
The three views were imported into Power BI (*Get Data → PostgreSQL
database*) and used to create:
- a bar chart of sites per parish
- a bar chart of sites per chronological period
- a map showing the location of each site, using `lat_dec` and `lon_dec`
  set as Latitude/Longitude data categories

## Files in this repository
- `Património_Arqueológico__pt__csv.csv` — original data
- `consultas.sql` — all SQL queries used in the project
- `painel_patrimonio.png` — screenshot of the final Power BI dashboard
- `patrimonio_arqueologico.pbix` — Power BI file
