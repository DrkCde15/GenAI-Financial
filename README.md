# GenAI-Financial

Pipeline de dados de criptomoedas construído no Databricks, seguindo a arquitetura medallion (camadas bronze, silver e gold). O projeto extrai cotações diárias do Bitcoin (BTC-USD) via Yahoo Finance, agrega por semana e padroniza os dados para análise.

## Estrutura do projeto

| Arquivo | Descrição |
| --- | --- |
| `camada_bronze` (notebook) | Ingestão de dados brutos: download de 1 ano de candles diários de BTC-USD via `yfinance`, com colunas renomeadas para português e carga em `bronze_bitcoin` |
| `camada_silver` (notebook) | Tratamento e agregação semanal (pandas): abertura/fechamento da semana, mínimo/máximo, volume médio e variação percentual; grava em `silver_bitcoin` |
| `camada_gold` (notebook) | Padronização final: lê a silver, converte datas para `date`, renomeia `variação` para `variacao_pct` e `volume` para `volume_medio`, e grava em `gold_bitcoin` |
| `01_setup` (query SQL) | Criação do banco `db_analytics` e das tabelas Delta `bronze_bitcoin` e `gold_bitcoin` (a `silver_bitcoin` é criada pelo notebook `camada_silver`) |

## Modelo de dados

* **`db_analytics.bronze_bitcoin`** — dados brutos por dia: `data` (DATE), `abertura`, `fechamento`, `minimo`, `maximo`, `volume` (DOUBLE)
* **`db_analytics.silver_bitcoin`** — agregação semanal intermediária: `semana_inicio`/`semana_fim` (timestamp), `abertura` (primeiro dia da semana), `fechamento` (último dia), `minimo`, `maximo`, `volume` (média semanal) e `variação` (variação percentual da semana, em %)
* **`db_analytics.gold_bitcoin`** — modelo semanal final: `semana_inicio`/`semana_fim` (DATE), `abertura`, `fechamento`, `minimo`, `maximo`, `variacao_pct` e `volume_medio` (DOUBLE)

## Como executar

1. Execute a query `01_setup` para criar o banco e as tabelas.
2. Execute o notebook `camada_bronze` (instala o `yfinance` automaticamente) — extrai os dados e carrega a `bronze_bitcoin`.
3. Execute o notebook `camada_silver` — agrega por semana e grava a `silver_bitcoin`.
4. Execute o notebook `camada_gold` — padroniza e grava a `gold_bitcoin`.

Todas as camadas gravam com `mode('overwrite')`, então o pipeline inteiro pode ser reexecutado de forma idempotente.

## Fonte de dados

Yahoo Finance (ticker `BTC-USD`), período de 1 ano, intervalo diário, via biblioteca `yfinance`.

