# GenAI-Financial

Pipeline de dados de criptomoedas construído no Databricks, seguindo a arquitetura medallion (camadas bronze, silver e gold). O projeto extrai cotações diárias do Bitcoin (BTC-USD) via Yahoo Finance, agrega por semana, padroniza os dados e, por fim, usa agentes de IA (LangGraph + Gemini) para gerar análise financeira comparativa e um report em HTML.

## Estrutura do projeto

| Arquivo | Descrição |
| --- | --- |
| `camada_bronze` (notebook) | Ingestão de dados brutos: download de 1 ano de candles diários de BTC-USD via `yfinance`, com colunas renomeadas para português e carga em `bronze_bitcoin` |
| `camada_silver` (notebook) | Tratamento e agregação semanal (pandas): abertura/fechamento da semana, mínimo/máximo, volume médio e variação percentual; grava em `silver_bitcoin` |
| `camada_gold` (notebook) | Padronização final: lê a silver, converte datas para `date`, renomeia `variação` para `variacao_pct` e `volume` para `volume_medio`, e grava em `gold_bitcoin` |
| `01_setup` (query SQL) | Criação do banco `db_analytics` e das tabelas Delta `bronze_bitcoin` e `gold_bitcoin` (a `silver_bitcoin` é criada pelo notebook `camada_silver`) |
| `agente_analise_report` (notebook) | Agentes com LangGraph: o **Agente 1** faz a análise financeira comparativa dos dados semanais e o **Agente 2** transforma a análise em um report HTML autocontido, salvo em `reports/` |
| `prompts/` | Prompts dos agentes versionados em arquivo: `agente1_analise_v1.md` (análise financeira) e `agente2_report_v1.md` (report HTML). Nova versão de prompt = novo arquivo (`_v2`, `_v3`...), sem quebrar as anteriores |
| `reports/` | Saída dos agentes — reports HTML gerados (ignorada pelo Git, pois é artefato gerado) |
| `.env` | Credenciais: `GEMINI_API_KEY` (Gemini) e `LANGCHAIN_API_KEY` (rastreamento no LangSmith). Ignorada pelo Git — nunca versionar |

## Modelo de dados

* **`db_analytics.bronze_bitcoin`** — dados brutos por dia: `data` (DATE), `abertura`, `fechamento`, `minimo`, `maximo`, `volume` (DOUBLE)
* **`db_analytics.silver_bitcoin`** — agregação semanal intermediária: `semana_inicio`/`semana_fim` (timestamp), `abertura` (primeiro dia da semana), `fechamento` (último dia), `minimo`, `maximo`, `volume` (média semanal) e `variação` (variação percentual da semana, em %)
* **`db_analytics.gold_bitcoin`** — modelo semanal final: `semana_inicio`/`semana_fim` (DATE), `abertura`, `fechamento`, `minimo`, `maximo`, `variacao_pct` e `volume_medio` (DOUBLE)

## Como executar

1. Execute a query `01_setup` para criar o banco e as tabelas.
2. Execute o notebook `camada_bronze` (instala o `yfinance` automaticamente) — extrai os dados e carrega a `bronze_bitcoin`.
3. Execute o notebook `camada_silver` — agrega por semana e grava a `silver_bitcoin`.
4. Execute o notebook `camada_gold` — padroniza e grava a `gold_bitcoin`.
5. Crie o arquivo `.env` na raiz do projeto com `GEMINI_API_KEY` (API do Gemini) e `LANGCHAIN_API_KEY` (LangSmith) — este arquivo é ignorado pelo Git.
6. Execute o notebook `agente_analise_report` — instala as dependências (`langgraph`, `langchain-google-genai`, `python-dotenv`), reinicia o Python e roda o pipeline dos dois agentes.

Todas as camadas gravam com `mode('overwrite')`, então o pipeline inteiro pode ser reexecutado de forma idempotente.

## Agentes de IA (LangGraph + Gemini)

O notebook `agente_analise_report` orquestra dois agentes encadeados em um grafo LangGraph:

```text
início → Agente 1 (análise financeira comparativa) → Agente 2 (report HTML) → fim
```

* **Agente 1 — análise financeira comparativa**: recebe os dados semanais da `gold_bitcoin` em JSON (histórico completo, comparativo das últimas 4 semanas vs. 4 anteriores, semanas em destaque) e produz a análise em markdown.
* **Agente 2 — geração de report**: recebe a análise do Agente 1 + os dados semanais e gera um report HTML autocontido (CSS embutido, sem dependências externas), salvo em `reports/relatorio_bitcoin_YYYYMMDD_HHMM.html`.

Detalhes de implementação:

* Modelo: `gemini-3.8-flash` via `langchain-google-genai` (`gemini-2.0-flash` foi descontinuado pela API).
* Prompts versionados em arquivo (pasta `prompts/`) — carregados como `system` message, permitindo evolução com versionamento explícito.
* Credenciais via `.env` (fora do Git); rastreamento das execuções no LangSmith (`LANGCHAIN_TRACING_V2=true`).
* A geração do HTML pode levar vários minutos; o LLM usa `timeout=600`.

## Fonte de dados

Yahoo Finance (ticker `BTC-USD`), período de 1 ano, intervalo diário, via biblioteca `yfinance`.

