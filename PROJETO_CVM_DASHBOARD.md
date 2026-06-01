# CVM Dashboard — Documentação do Projeto

## Visão geral
Dashboard web que cruza movimentações de insiders (diretores, conselheiros, controladores)
declaradas à CVM com a cotação histórica da ação na B3, exibindo um gráfico candlestick
com marcações dos dias em que houve compra ou venda à vista.

---

## Repositórios GitHub

| Repo | Visibilidade | Função |
|---|---|---|
| `Talharino/robo-cvm` | Privado | Robô de coleta CVM + IA (fase 2) |
| `Talharino/CVM-dashboard` | Público | Site do dashboard (GitHub Pages) |

**URL do site:** `https://talharino.github.io/CVM-dashboard`

---

## Supabase

**Projeto:** `meu-app-cvm`
**ID:** `uqadyxirtmeqybgobmai`
**URL:** `https://uqadyxirtmeqybgobmai.supabase.co`
**Região:** `sa-east-1`

### Tabelas

| Tabela | Linhas | Descrição |
|---|---|---|
| `CIA_ABERTA_DOC_VLMO_DADOS` | ~40.656 | Documentos IPE da CVM (metadados) |
| `MOVIMENTACOES` | ~12.780 | Movimentações de valores mobiliários |
| `TICKER_MAPPING` | ~130 | De-para: codigo_cvm → ticker B3 |

### Função SQL
```sql
get_movimentacoes_insider(p_codigo_cvm, p_from, p_grupo)
```
Faz o JOIN entre MOVIMENTACOES e CIA_ABERTA_DOC_VLMO_DADOS internamente,
filtrando por `realizadas_operacoes = 'Sim'` e `operacao IN ('Compra à vista', 'Venda à vista')`.

### RLS
- `CIA_ABERTA_DOC_VLMO_DADOS` — RLS ativo, escrita para authenticated
- `MOVIMENTACOES` — RLS ativo, insert para authenticated
- `TICKER_MAPPING` — RLS ativo, SELECT para anon+authenticated, escrita para authenticated

---

## Edge Functions (Supabase)

### `/cotacao`
**URL:** `https://uqadyxirtmeqybgobmai.supabase.co/functions/v1/cotacao`
**Auth:** `verify_jwt = false` (pública)

**Parâmetros:**
- `ticker` (obrigatório) — ex: `ABEV3` ou `ABEV3.SA`
- `from` (opcional) — data inicial, padrão `2026-01-01`

**Retorno:**
```json
{
  "ticker": "ABEV3.SA",
  "candles": [
    { "date": "2026-01-02", "open": 13.20, "high": 13.45, "low": 13.10, "close": 13.30, "volume": 12345678 }
  ]
}
```
**Fonte:** Yahoo Finance (`query1.finance.yahoo.com`) — dados reais e gratuitos.

---

### `/movimentacoes`
**URL:** `https://uqadyxirtmeqybgobmai.supabase.co/functions/v1/movimentacoes`
**Auth:** `verify_jwt = false` (pública)

**Parâmetros:**
- `cvm` (obrigatório) — código CVM da empresa, ex: `23264`
- `from` (opcional) — data inicial, padrão `2026-01-01`
- `grupo` (opcional) — filtro por grupo, ex: `Diretoria`, `Conselho`, `Controlador`

**Retorno:**
```json
{
  "codigo_cvm": "23264",
  "empresa": "AMBEV S.A.",
  "ticker_b3": "ABEV3",
  "ticker_yahoo": "ABEV3.SA",
  "total_movimentacoes": 53,
  "totais": { "compra": 2, "venda": 51, "vol_total": 26100000 },
  "movimentacoes": [...],
  "por_dia": {
    "2026-01-05": { "compra": 1, "venda": 0, "vol_compra": 1058687, "vol_venda": 0, "qtd_compra": 76472, "qtd_venda": 0 }
  }
}
```

---

## GitHub Actions (robo-cvm)

### `rotina.yml` — Rotina Mensal CVM
- Roda dia 20 de cada mês às 23h
- Fase 1: `coletor.py` — baixa CSV da CVM e faz upsert no Supabase
- Fase 2: `fase2_ia_tabulacao.py` — lê PDFs e extrai dados com Claude (ANTHROPIC_API_KEY)

### `atualizar_tickers.yml` — Atualização de Tickers
- Roda nos dias 28-31 de cada mês (verifica internamente se é o último dia)
- Script: `popular_ticker_mapping.py`
- Busca `codigo_cvm` de todas as empresas no Supabase
- Busca ticker B3 no `cnpjaberto.com.br` por código CVM
- Faz upsert na tabela `TICKER_MAPPING`
- **Atenção:** a busca por `?cvm=XXXXX` no cnpjaberto retornou resultados errados (AERI3 para todas).
  A tabela foi populada manualmente com ~130 tickers corretos. O script precisa ser corrigido
  para buscar pelo ticker direto (lista fixa) em vez de pelo CVM.

### Secrets configurados
```
SUPABASE_URL      → https://uqadyxirtmeqybgobmai.supabase.co
SUPABASE_KEY      → anon key (JWT)
ANTHROPIC_API_KEY → chave Claude
ROBO_EMAIL        → email do usuário robô no Supabase Auth
ROBO_PASSWORD     → senha do usuário robô
GEMINI_API_KEY    → (existente, não usado neste projeto)
```

---

## Dashboard (CVM-dashboard)

### Arquivo: `index.html`
Single-page app em HTML/CSS/JS puro. Sem frameworks. Sem build.

**Funcionalidades:**
- Busca por código CVM ou nome da empresa (autocomplete via Supabase REST)
- Filtro por grupo (Diretoria, Conselho, Controlador, Conselho Fiscal)
- Cards de métricas: empresa, ticker, totais, volume total
- Cards separados compra/venda com: operações, quantidade, volume financeiro, preço médio
- Gráfico candlestick diário (canvas 2D puro) com cotação real do Yahoo Finance
  - Candle branco = alta (close > open)
  - Candle preto = baixa (close < open)
  - Seta verde ▲ abaixo do candle = dia com compra insider
  - Seta vermelha ▼ acima do candle = dia com venda insider
  - Tooltip com OHLC + dados da operação ao passar o mouse
- Gráfico de barras com volume financeiro das operações por dia (Chart.js)

---

## Pendências / Próximos passos

1. **Corrigir script `popular_ticker_mapping.py`** — a busca por CVM no cnpjaberto não funciona.
   Solução: usar lista de tickers conhecidos e buscar o CVM na página do ticker (como era antes),
   ou manter a tabela populada manualmente e só atualizar quando necessário.

2. **Completar TICKER_MAPPING** — ~130 empresas mapeadas manualmente, mas há ~200 com
   movimentações em 2026. Empresas sem ticker (SPEs, securitizadoras) não têm cotação — comportamento esperado.

3. **Evoluir para app mobile** — arquitetura recomendada:
   - Manter Edge Functions como backend
   - Substituir Yahoo Finance por Brapi (`brapi.dev`) para produção com 5k usuários
   - Frontend React Native ou Flutter consumindo as mesmas Edge Functions

4. **Adicionar período configurável** — atualmente fixo em 01/01/2026.

5. **Adicionar mais grupos de insiders** no filtro conforme necessidade.

---

## Como retomar em nova sessão Claude

"Tenho um projeto CVM Dashboard. Site em https://talharino.github.io/CVM-dashboard,
Supabase ID uqadyxirtmeqybgobmai, duas Edge Functions: /cotacao (Yahoo Finance) e
/movimentacoes (dados CVM). Robô de coleta no repo privado robo-cvm. Consulte o arquivo
PROJETO_CVM_DASHBOARD.md no repo para detalhes completos."
