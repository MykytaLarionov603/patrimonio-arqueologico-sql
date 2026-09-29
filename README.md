# patrimonio-arqueologico-sql

# Património Arqueológico — Concelho de Águeda

Projeto de análise de dados sobre o património arqueológico do concelho de Águeda,
usando PostgreSQL para tratar os dados e Power BI para os visualizar num painel
interativo com gráficos e mapa.

## Fonte dos dados
Câmara Municipal de Águeda — Portal de Dados Abertos
https://dados.gov.pt/organizations/camara-municipal-de-agueda

## Ferramentas usadas
- **PostgreSQL + pgAdmin** — importação, tratamento e consulta dos dados
- **Power BI** — visualização, gráficos e mapa interativo

## Pergunta do projeto
**Como se distribui o património arqueológico do concelho por freguesia e por período cronológico, e onde se localiza no mapa?**

Para responder, foram feitas três análises:
1. Número de sítios por freguesia
2. Número de sítios por período cronológico
3. Localização de cada sítio num mapa

## O que descobri
- 17 sítios arqueológicos identificados no concelho de Águeda
- Espinhel é a freguesia com mais sítios (4 de 17)
- Por período cronológico, os sítios de época Romana e os de cronologia
  Indeterminada empatam como os mais comuns (5 sítios cada), seguidos da
  Pré-história (4 sítios) e do período Medieval (3 sítios)
- A classificação por período é aproximada: o campo `cronologia` original
  vem em texto livre (ex: "Romano (?)", "Alta Idade Media", "Tardo
  romano/medieval") e foi simplificada com uma regra baseada em palavras-
  chave, que pode falhar em casos de variação de palavra (ex: "Romana" em
  vez de "Romano" não foi reconhecido nessa categoria)

## Processo — passo a passo

### 1. Importação do CSV para o PostgreSQL
Criação da tabela e importação do ficheiro CSV original através do pgAdmin
(*Import/Export Data*), com o campo **Escape** definido como `"` em vez do
valor por defeito `'`, para lidar corretamente com as plicas presentes nas
coordenadas (ex: `40º 39' 12.41" N`).

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

### 2. Exploração dos dados
Consultas iniciais para conhecer a tabela: total de registos, freguesias
existentes, filtros por cronologia e por tipo de sítio.

```sql
SELECT count(*) FROM pa_test;

SELECT DISTINCT freguesia FROM pa_test;

SELECT ident_part FROM pa_test WHERE ident_part ILIKE '%mamoa%';
```

### 3. Sítios por freguesia

```sql
SELECT freguesia, count(*) AS total
FROM pa_test
GROUP BY freguesia
ORDER BY total DESC;
```

### 4. Sítios por período cronológico
A coluna `cronologia` tinha muitos valores diferentes em texto livre, por
isso foram agrupados em categorias com `CASE`:

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

### 5. Conversão das coordenadas
As coordenadas vinham em graus/minutos/segundos (ex: `40º 39' 12.41" N`) e
foram convertidas para graus decimais, formato exigido pelo Power BI para
desenhar o mapa:

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

### 6. Views criadas para o Power BI
Em vez de importar a tabela toda, foram criadas *views* já com o resultado
de cada análise:

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

### 7. Ligação ao Power BI
As três *views* foram importadas no Power BI (*Obter dados → Base de dados
PostgreSQL*) e usadas para criar:
- um gráfico de barras com os sítios por freguesia
- um gráfico de barras com os sítios por período cronológico
- um mapa com a localização de cada sítio, usando `lat_dec` e `lon_dec`
  classificados como categorias de dados Latitude/Longitude

## Ficheiros neste repositório
- `Património_Arqueológico__pt__csv.csv` — dados originais
- `consultas.sql` — todas as consultas SQL usadas no projeto
- `painel_patrimonio.png` — captura de ecrã do painel final no Power BI
- `patrimonio_arqueologico.pbix` — ficheiro do Power BI 
