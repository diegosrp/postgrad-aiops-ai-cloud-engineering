# Questão 03 - Relatório de redução de custos cloud

## Prompt

```text
# Task
Gerar um relatório executivo-técnico de redução de custos cloud com base no CSV abaixo.

servico,categoria,custo_mensal_usd,uso_medio_pct,observacao
EC2 reservada,compute,4200,72,contrato de 1 ano
EC2 on-demand,compute,8200,45,workloads variaveis
EKS,compute,6700,58,3 clusters
RDS PostgreSQL,databases,8200,62,multi-AZ
ElastiCache Redis,databases,2100,40,cluster de producao
S3 Standard,storage,3100,,5 buckets principais
EBS gp3,storage,1600,68,volumes de producao
CloudWatch Logs,observability,2800,,retencao de 90 dias
CloudWatch Metrics,observability,900,,
Data Transfer Out,network,1900,,trafego entre regioes
NAT Gateway,network,1200,,3 gateways ativos
Lambda,compute,900,30,~12M invocacoes/mes

# Action
1. Calcule o custo total mensal.
2. Identifique oportunidades de economia priorizadas por impacto financeiro.
3. Para cada oportunidade, traga: economia estimada em USD/mês, percentual sobre a conta total, esforço de implementação (baixo/médio/alto), riscos/pré-requisitos.
4. Monte um plano em ondas (30, 60 e 90 dias) com quick wins primeiro.
5. Inclua uma conclusão dizendo se a meta trimestral de 15% é atingível com o plano proposto.

# Goal
Entregar um relatório pronto para apresentação à diretoria, orientado a atingir pelo menos 15% de redução de custo cloud sem degradar SLA.

# Format
- Resumo executivo (5 a 8 linhas).
- Tabela priorizada de oportunidades.
- Plano em ondas (30/60/90 dias).
- Conclusão objetiva sobre meta de 15%.
```

## Modelo

OpenAI GPT-5.4. Escolhi este modelo pela capacidade de correlacionar dados tabulares com raciocínio analítico e transformar isso em recomendações priorizadas, com equilíbrio entre precisão numérica e comunicação executiva.

## Output

```markdown
### Resumo executivo
O custo mensal total atual é de **USD 41.800**. Os maiores concentradores de gasto estão em compute e banco de dados, com sinais claros de subutilização em EC2 on-demand, EKS, ElastiCache Redis e Lambda. A estratégia recomendada é capturar primeiro ganhos de configuração e retenção, depois avançar para compromissos de capacidade e ajustes estruturais. As oportunidades priorizadas somam **USD 7.240/mês** em economia potencial, o equivalente a **17,3%** da conta. O plano em ondas foi desenhado para preservar SLA, com quick wins nos primeiros 30 dias e mudanças de maior impacto distribuídas ao longo do trimestre. Com governança de execução e validação por métricas de capacidade, a meta trimestral de 15% é atingível.

### Oportunidades priorizadas
| Prioridade | Oportunidade | Economia (USD/mês) | % da conta total | Esforço | Riscos / Pré-requisitos |
|---|---|---:|---:|---|---|
| 1 | Rightsizing de EC2 on-demand com contratação parcial de Savings Plan | 2.460 | 5,9% | Médio | Exige análise de sazonalidade, baseline de CPU e memória e cobertura mínima para picos |
| 2 | Otimização de RDS PostgreSQL com reserva parcial de capacidade | 1.640 | 3,9% | Médio | Requer janela de manutenção, validação de IOPS e teste de performance após ajuste |
| 3 | Redução da retenção de CloudWatch Logs de 90 para 30 ou 45 dias por classe | 1.260 | 3,0% | Baixo | Precisa alinhar retenção com compliance, auditoria e times consumidores de log |
| 4 | Consolidação de nós EKS com ajuste de requests/limits e autoscaling | 1.005 | 2,4% | Alto | Necessita revisão de capacidade por cluster e teste controlado em workloads críticos |
| 5 | Otimização de ElastiCache Redis por rightsizing e revisão de réplicas | 420 | 1,0% | Médio | Validar latência, hit ratio e necessidade real de capacidade provisionada |
| 6 | Redução de custo de NAT Gateway e tráfego inter-região | 455 | 1,1% | Médio | Depende de revisão de rotas, afinidade regional e possíveis ajustes de arquitetura |

**Economia total estimada:** USD 7.240/mês (**17,3%**)

### Plano em ondas
- **30 dias:** reduzir retenção de CloudWatch Logs, revisar rotas com impacto em NAT Gateway e tráfego inter-região e levantar baseline de uso de EC2 e Redis para capturar quick wins sem mudança arquitetural.
- **60 dias:** executar rightsizing de EC2 on-demand, contratar Savings Plan parcial para carga estável e aplicar ajustes seguros em ElastiCache com acompanhamento de latência e consumo.
- **90 dias:** concluir otimização de RDS PostgreSQL, consolidar capacidade do EKS com revisão de requests e limits e instituir rotina mensal de governança FinOps.

### Conclusão
A meta trimestral de **15%** é **atingível**. O plano proposto oferece potencial de **17,3%** de redução mensal, com margem suficiente para absorver variações de execução e ainda manter o objetivo acima da meta, desde que cada onda seja acompanhada por métricas de performance, capacidade e risco operacional.
```

## Justificativa

Task define claramente o problema de negócio (reduzir custos com base no CSV fornecido). Action detalha o método de execução da análise, incluindo cálculo, priorização, esforço e riscos. Goal fixa a meta mensurável de 15% sem comprometer SLA, direcionando a resposta para impacto real. O T-A-G foi adequado porque o sucesso depende diretamente de KPI financeiro.
