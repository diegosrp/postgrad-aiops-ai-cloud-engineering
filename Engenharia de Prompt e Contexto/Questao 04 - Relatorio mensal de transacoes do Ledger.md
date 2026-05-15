# Questão 04 - Relatório mensal de transações do Ledger

## Prompt

```text
# Task
Escrever uma query SQL PostgreSQL para consolidar transações do Ledger nos últimos 6 meses corridos a partir de 2026-04-24.

# Action
1. Considere o schema completo abaixo (PostgreSQL) como fonte de verdade:
CREATE TABLE transactions (
  id              BIGSERIAL PRIMARY KEY,
  customer_id     BIGINT NOT NULL REFERENCES customers(id),
  category        VARCHAR(32) NOT NULL,
  amount_cents    BIGINT NOT NULL,
  status          VARCHAR(16) NOT NULL,
  payment_method  VARCHAR(16),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  completed_at    TIMESTAMPTZ
);


CREATE INDEX idx_transactions_created_at ON transactions(created_at);
CREATE INDEX idx_transactions_status ON transactions(status);
CREATE INDEX idx_transactions_category ON transactions(category);


CREATE TABLE customers (
  id          BIGSERIAL PRIMARY KEY,
  segment     VARCHAR(16) NOT NULL,
  country     CHAR(2) NOT NULL,
  signup_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);


2. Usar apenas a tabela transactions para agregação principal, com status = 'completed' (sem JOIN com customers para os cálculos).
3. Considerar somente as categorias: subscription, one_time, refund, credit_adjustment.
4. Filtrar período dos últimos 6 meses corridos a partir de 2026-04-24.
5. Agrupar por mês (formato YYYY-MM) e por categoria.
6. Retornar por linha: quantidade de transações e volume total em reais (amount_cents / 100.0 com 2 casas decimais).
7. Ordenar por mês crescente e categoria crescente.
8. Garantir SQL legível e pronta para uso em produção.

# Goal
Entregar uma query correta e executável que Jennifer possa usar diretamente no fechamento mensal para apresentação executiva, sem retrabalho manual.

# Format
- Somente um bloco de código SQL.
```

## Modelo

OpenAI GPT-5.4. Escolhi este modelo pela precisão em SQL analítico e pela aderência estrita a filtros, tipos de dados, agrupamentos e ordenação, reduzindo a chance de ajustes manuais na query final.

## Output

```sql
WITH params AS (
  SELECT
    DATE '2026-04-24' AS anchor_date,
    DATE '2026-04-24' - INTERVAL '6 months' AS start_date,
    DATE '2026-04-24' + INTERVAL '1 day' AS end_date
)
SELECT
  TO_CHAR(DATE_TRUNC('month', t.created_at), 'YYYY-MM') AS month,
  t.category,
  COUNT(*) AS transaction_count,
  ROUND(SUM(t.amount_cents) / 100.0, 2) AS total_volume_brl
FROM transactions AS t
CROSS JOIN params AS p
WHERE t.status = 'completed'
  AND t.category IN ('subscription', 'one_time', 'refund', 'credit_adjustment')
  AND t.created_at >= p.start_date
  AND t.created_at < p.end_date
GROUP BY
  DATE_TRUNC('month', t.created_at),
  t.category
ORDER BY
  DATE_TRUNC('month', t.created_at) ASC,
  t.category ASC;
```

## Justificativa

Task estabelece a entrega principal: query SQL para o recorte de 6 meses. Action agora inclui o schema completo com CREATE TABLE para transactions e customers, além dos passos objetivos de implementação (filtro por status/categoria, agrupamento mensal, métricas e ordenação), aumentando confiabilidade e reprodutibilidade. Goal garante foco no uso real da área de negócio, com saída executável sem ajustes manuais. O T-A-G encaixa porque há um resultado operacional claro ligado à tomada de decisão.
