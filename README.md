# GenAI-Financial

Pipeline de dados de criptomoedas construído no Databricks, seguindo a arquitetura medallion (camadas bronze e gold). O projeto extrai cotações diárias do Bitcoin (BTC-USD) via Yahoo Finance e prepara os dados para análise semanal.

## Estrutura do projeto

| Arquivo | Descrição |
| --- | --- |
| `camada_bronze` (notebook) | Ingestão de dados brutos: download de 1 ano de candles diários de BTC-USD via `yfinance`, com colunas renomeadas para português (`data`, `abertura`, `fechamento`, `minimo`, `maximo`, `volume`) |
| `01_setup` (query SQL) | Criação do banco `db_analytics` e das tabelas Delta `bronze_bitcoin` e `gold_bitcoin` |

## Modelo de dados

* **`db_analytics.bronze_bitcoin`** — dados brutos granulares por dia: `data`, `abertura`, `fechamento`, `minima`, `maxima`, `volume`
* **`db_analytics.gold_bitcoin`** — dados agregados por semana: `semana_inicio`, `semana_fim`, `abertura`, `fechamento`, `minima`, `maxima`, `variacao_pct` (variação percentual semanal) e `volume_medio`

## Como executar

1. Execute a query `01_setup` para criar o banco e as tabelas.
2. Execute o notebook `camada_bronze` (instala o `yfinance` automaticamente) para extrair os dados de origem.
3. (Próximo passo) Carregar o DataFrame `dados_bronze` na tabela `bronze_bitcoin` e construir a camada gold com a agregação semanal.

## Fonte de dados

Yahoo Finance (ticker `BTC-USD`), período de 1 ano, intervalo diário, via biblioteca `yfinance`.

