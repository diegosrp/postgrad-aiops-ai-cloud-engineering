# Questão 04 - Relatório mensal de transações do Ledger

## Prompt

```text
# Task
Escrever uma query SQL PostgreSQL para consolidar transações do Ledger nos últimos 6 meses corridos a partir de 2026-04-24.

# Action
1. Usar apenas a tabela transactions para agregação principal, com status = 'completed'.
2. Considerar somente as categorias: subscription, one_time, refund, credit_adjustment.
3. Filtrar período dos últimos 6 meses corridos a partir de 2026-04-24.
4. Agrupar por mês (formato YYYY-MM) e por categoria.
5. Retornar por linha: quantidade de transações e volume total em reais (amount_cents / 100.0 com 2 casas decimais).
6. Ordenar por mês crescente e categoria crescente.
7. Garantir SQL legível e pronta para uso em produção.

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
    DATE '2026-04-24' AS ref_date,
    (DATE '2026-04-24' - INTERVAL '6 months') AS start_date
),
base AS (
  SELECT
    date_trunc('month', t.completed_at)::date AS month_start,
    t.category,
    t.amount_cents
  FROM transactions t
  CROSS JOIN params p
  WHERE t.status = 'completed'
    AND t.category IN ('subscription', 'one_time', 'refund', 'credit_adjustment')
    AND t.completed_at >= p.start_date
    AND t.completed_at < (p.ref_date + INTERVAL '1 day')
)
SELECT
  to_char(month_start, 'YYYY-MM') AS month,
  category,
  COUNT(*) AS transaction_count,
  ROUND(SUM(amount_cents) / 100.0, 2) AS total_brl
FROM base
GROUP BY month_start, category
ORDER BY month_start ASC, category ASC;
```

## Justificativa

Task estabelece a entrega principal: query SQL para o recorte de 6 meses. Action transforma o requisito em passos objetivos de implementação (filtro por status/categoria, agrupamento mensal, métricas e ordenação). Goal garante foco no uso real da área de negócio, com saída executável sem ajustes manuais. O T-A-G encaixa porque há um resultado operacional claro ligado à tomada de decisão.
