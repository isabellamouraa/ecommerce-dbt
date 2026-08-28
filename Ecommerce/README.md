# 🛒 E-commerce Analytics — Camada Analítica dbt + Dashboard de BI

Projeto de **Engenharia e Análise de Dados** que implementa uma camada analítica completa sobre um banco de dados PostgreSQL de e-commerce, utilizando **dbt (Data Build Tool)** com **Arquitetura Medalhão** (Bronze → Silver → Gold) e um **Dashboard executivo de Business Intelligence** em HTML puro.

---

## 📐 Arquitetura Geral

```
PostgreSQL (Supabase)
└── Schema public (tabelas raw)
    ├── raw.vendas
    ├── raw.clientes
    ├── raw.produtos
    └── raw.preco_competidores
         │
         ▼
    ┌──────────────────────────────────────────────────────┐
    │               Arquitetura Medalhão (dbt)             │
    │                                                      │
    │  BRONZE (4 views)   ──►  SILVER (4 tables)          │
    │  Cópia fiel das            Colunas calculadas        │
    │  tabelas raw               (sem JOINs, sem filtros)  │
    │                                    │                 │
    │                                    ▼                 │
    │                         GOLD (3 Data Marts)          │
    │                         JOINs + agregações           │
    │                         + regras de negócio          │
    └──────────────────────────────────────────────────────┘
         │
         ▼
    Dashboard de BI (index.html)
    Visualização interativa dos Data Marts Gold
```

---

## 🗂️ Estrutura do Projeto

```
models/
├── _sources.yml                   → Define as 4 tabelas raw no schema public
├── bronze/
│   ├── bronze_vendas.sql          → view: cópia de raw.vendas
│   ├── bronze_clientes.sql        → view: cópia de raw.clientes
│   ├── bronze_produtos.sql        → view: cópia de raw.produtos
│   └── bronze_preco_competidores.sql
├── silver/
│   ├── silver_vendas.sql          → table: + colunas de data/hora, receita_total
│   ├── silver_clientes.sql        → table: pass-through
│   ├── silver_produtos.sql        → table: + faixa_preco (PREMIUM/MEDIO/BASICO)
│   └── silver_preco_competidores.sql → table: + data_coleta_date
└── gold/
    ├── sales/
    │   └── vendas_temporais.sql   → Data Mart de Vendas (receita, pedidos, ticket médio por período/canal/hora)
    ├── customer_success/
    │   └── clientes_segmentacao.sql → Data Mart de Clientes (segmentação VIP/TOP TIER/REGULAR + ranking LTV)
    └── pricing/
        └── precos_competitividade.sql → Data Mart de Preços (posicionamento vs concorrentes)

index.html                         → Dashboard de BI executivo (interativo, autocontido)
prd.md                             → Product Requirements Document da camada analítica
dbt_project.yml                    → Configuração do projeto dbt
```

---

## 🧱 Camadas do Projeto

### 🟫 Bronze — Contrato de Ingestão
- **Materialização**: `view`
- **Schema**: `bronze`
- **Regra**: SELECT explícito de todas as colunas da fonte. **Zero transformações**, **zero filtros**.
- Serve como contrato formal entre a camada raw e o pipeline de transformação.

### 🔘 Silver — Padronização & Enriquecimento
- **Materialização**: `table`
- **Schema**: `silver`
- Relação 1:1 com o Bronze (sem JOINs entre tabelas).
- Adiciona **colunas calculadas derivadas**:

| Modelo | Colunas Adicionadas |
|---|---|
| `silver_vendas` | `receita_total = quantidade * preco_unitario`, `data_venda_date`, `ano_venda`, `mes_venda`, `dia_venda`, `dia_semana`, `hora_venda` |
| `silver_produtos` | `faixa_preco` (`PREMIUM` > R$ 1.000 / `MEDIO` > R$ 500 / `BASICO` ≤ R$ 500) |
| `silver_preco_competidores` | `data_coleta_date` |
| `silver_clientes` | pass-through (sem transformações) |

### 🟡 Gold — Data Marts para Consumo de BI
- **Materialização**: `table`
- **Schema**: `gold`
- JOINs estratégicos entre tabelas Silver + agregações + regras de negócio.
- 1 modelo por Data Mart, organizados em subpastas.

---

## 📊 Data Marts (Camada Gold)

### 1. `vendas_temporais` (Sales)
> **Pergunta respondida:** Como estão as vendas ao longo do tempo e por canal?

Agrega as vendas da `silver_vendas` por data, com métricas diárias:

| Métrica | Fórmula |
|---|---|
| `receita_total` | `SUM(quantidade * preco_unitario)` |
| `quantidade_total` | `SUM(quantidade)` |
| `total_vendas` | `COUNT(DISTINCT id_venda)` |
| `total_clientes_unicos` | `COUNT(DISTINCT id_cliente)` |
| `ticket_medio` | `AVG(receita_total)` |

Dimensões temporais disponíveis: `data_venda_date`, `ano_venda`, `mes_venda`, `dia_venda`, `dia_semana_nome` (Domingo a Sábado), `hora_venda`.

---

### 2. `clientes_segmentacao` (Customer Success)
> **Pergunta respondida:** Quais são meus melhores clientes?

JOIN entre `silver_vendas` e `silver_clientes`, segmentando clientes por LTV acumulado:

| Segmento | Regra (variável dbt) |
|---|---|
| **VIP** | `receita_total >= R$ 10.000` (`var('segmentacao_vip_threshold', 10000)`) |
| **TOP_TIER** | `receita_total >= R$ 5.000` (`var('segmentacao_top_tier_threshold', 5000)`) |
| **REGULAR** | `receita_total < R$ 5.000` |

Colunas de saída: `cliente_id`, `nome_cliente`, `estado`, `receita_total`, `total_compras`, `ticket_medio`, `primeira_compra`, `ultima_compra`, `segmento_cliente`, `ranking_receita`.

---

### 3. `precos_competitividade` (Pricing)
> **Pergunta respondida:** Como estamos em relação à concorrência?

JOIN entre `silver_produtos`, `silver_preco_competidores` e `silver_vendas`.

Classificação de competitividade de preço (avaliada nesta ordem):

| Classificação | Condição |
|---|---|
| `MAIS_CARO_QUE_TODOS` | `nosso_preco > preco_maximo_concorrentes` |
| `MAIS_BARATO_QUE_TODOS` | `nosso_preco < preco_minimo_concorrentes` |
| `ACIMA_DA_MEDIA` | `nosso_preco > preco_medio_concorrentes` |
| `ABAIXO_DA_MEDIA` | `nosso_preco < preco_medio_concorrentes` |
| `NA_MEDIA` | demais casos |

Colunas calculadas: `diferenca_percentual_vs_media`, `diferenca_percentual_vs_minimo`, `receita_total`, `quantidade_total` (com COALESCE para produtos sem vendas).

---

## 📈 Dashboard de BI (`index.html`)

Dashboard executivo **100% autocontido em HTML/CSS/JavaScript**, sem dependências de servidor. Basta abrir no navegador.

### Funcionalidades

| Recurso | Detalhe |
|---|---|
| **5 abas de navegação** | Visão Geral, Vendas, Customer Success, Pricing, Arquitetura dbt |
| **Filtros globais** | Período de análise, Canal de venda, Categoria do produto, Segmento de cliente |
| **11 gráficos interativos** | Chart.js — Linha/Área, Barras, Donut |
| **2 tabelas analíticas** | Busca, ordenação por coluna, paginação e badges de status |
| **Exportação CSV** | Clientes filtrados exportados diretamente |
| **Dark / Light Mode** | Alternância de tema com persistência visual |
| **Responsivo** | Desktop, Notebook, Tablet e Mobile |

### Motor Analítico (In-Memory)
O dashboard simula o pipeline dbt diretamente no browser: os dados de demonstração são gerados com um gerador pseudoaleatório determinístico e todas as transformações das camadas Silver e Gold (faixas de preço, segmentação de clientes, classificação de competitividade) são aplicadas em JavaScript, seguindo **exatamente** as mesmas regras SQL do `prd.md`.

---

## ⚙️ Configuração dbt

```yaml
# dbt_project.yml
models:
  ecommerce:
    bronze:
      +materialized: view
      +schema: bronze
    silver:
      +materialized: table
      +schema: silver
    gold:
      +materialized: table
      +schema: gold

vars:
  segmentacao_vip_threshold: 10000
  segmentacao_top_tier_threshold: 5000
```

---

## 🚀 Como Executar

```bash
# Instalar dependências
pip install dbt-postgres

# Configurar o profiles.yml com as credenciais do Supabase
# Executar todos os modelos
dbt run

# Executar apenas a camada Gold
dbt run --select gold

# Executar testes
dbt test
```

### Dashboard de BI
Abra o arquivo `index.html` diretamente no navegador — sem servidor necessário.

---

## 🗃️ Fontes de Dados (raw)

| Tabela | Descrição | Colunas Principais |
|---|---|---|
| `raw.vendas` | Transações de venda | `id_venda`, `data_venda`, `id_cliente`, `id_produto`, `canal_venda`, `quantidade`, `preco_unitario` |
| `raw.clientes` | Cadastro de clientes | `id_cliente`, `nome_cliente`, `estado`, `pais`, `data_cadastro` |
| `raw.produtos` | Catálogo de produtos | `id_produto`, `nome_produto`, `categoria`, `marca`, `preco_atual` |
| `raw.preco_competidores` | Coleta de preços de concorrentes | `id_produto`, `nome_concorrente`, `preco_concorrente`, `data_coleta` |

---

## 🛠️ Stack Técnica

| Componente | Tecnologia |
|---|---|
| Banco de Dados | PostgreSQL (Supabase) |
| Transformação | dbt Core |
| Linguagem SQL | PostgreSQL dialect |
| Dashboard | HTML5 + CSS3 + JavaScript (ES6+) |
| Gráficos | Chart.js (CDN) |
| Tipografia | Google Fonts — Plus Jakarta Sans |

---

## 📁 Entregáveis

| Entregável | Quantidade | Tipo |
|---|---|---|
| Modelos Bronze | 4 | `view` |
| Modelos Silver | 4 | `table` |
| Modelos Gold (Data Marts) | 3 | `table` |
| `_sources.yml` | 1 | YAML |
| `dbt_project.yml` | 1 | YAML |
| Dashboard de BI | 1 | HTML autocontido |
| **Total** | **14** | |
