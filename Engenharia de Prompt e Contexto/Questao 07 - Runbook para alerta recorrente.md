# Questão 07 - Runbook para alerta recorrente

## Prompt

```text
# Role
Você é um SRE sênior de plantão, especialista em EKS, troubleshooting de aplicações distribuídas e resposta a incidentes com foco em padronização operacional.

# Input
- Alerta recorrente: [CRITICAL] High memory usage on Chronos API pods (>85% for 10min).
- Ambiente: EKS, namespace production, 6 réplicas com HPA (min 4, max 12, CPU target 70%).
- Deploy via Argo CD (repo hvt/chronos-api).
- Dependências: Ledger (PostgreSQL) e Reactor (SQS).
- Observabilidade: /metrics, logs no Beacon, dashboards no Grafana.
- Ferramentas do plantão: kubectl, aws cli, argocd cli.
- Canal Slack: #oncall-chronos.
- Escalação: @chronos-core (SLA 15 min comercial, 30 min fora).

# Steps
1. Defina checklist inicial de triagem (5 minutos), com comandos específicos.
2. Crie fluxo de diagnóstico sequencial (memória por pod, eventos de OOM, limites/requests, dependências externas, versão de deploy).
3. Em cada passo, inclua: comando, o que verificar no resultado e decisão seguinte.
4. Defina critérios objetivos para escalar ao @chronos-core.
5. Defina ações de mitigação imediata de baixo risco.
6. Defina critério de encerramento do incidente e checklist pós-incidente.

# Expectation
Entregar um runbook procedural completo, reproduzível por qualquer plantonista, em linguagem operacional, com comandos prontos para copiar e colar e critérios de decisão sem ambiguidade.

# Format
- Runbook numerado em etapas.
- Para cada etapa: Comando, Verificação esperada, Próxima ação.
- Seção de Escalação.
- Seção de Encerramento.
```

## Modelo

Anthropic Claude 4.6 Sonnet. Escolhi este modelo pela qualidade em estruturar troubleshooting sequencial, com decisões condicionais claras e comandos operacionais prontos para execução.

## Output

```markdown
## Runbook: [CRITICAL] High memory usage on Chronos API pods (>85% for 10min)

Ambiente: EKS · namespace production · HPA min 4 / max 12 · Argo CD hvt/chronos-api
Slack: #oncall-chronos · Escalação: @chronos-core (SLA 15 min comercial / 30 min fora)

---

### 1. Triagem inicial (até 5 min)

Comandos:

- `kubectl -n production get pods -l app=chronos-api -o wide`
- `kubectl -n production top pods -l app=chronos-api --containers`
- `kubectl -n production get hpa chronos-api`

Verificação esperada: identificar quais pods estão com memória >85% e se HPA já atingiu maxReplicas (12).

Próxima ação:

- Múltiplos pods >85% e HPA no teto → ir direto à etapa 2.
- Pod único isolado → verificar reinícios recentes antes de avançar.

---

### 2. Confirmar OOM e reinícios por memória

Comandos:

- `kubectl -n production describe pod <pod_name> | grep -A5 -E "OOM|Reason|Last State"`
- `kubectl -n production get events --sort-by=.lastTimestamp | grep -i "oom\|kill\|evict" | tail -20`

Verificação esperada: presença de OOMKilled, Reason: OOMKilled ou Evicted nos eventos recentes.

Próxima ação:

- OOM confirmado → abrir incidente, notificar #oncall-chronos e seguir etapa 5.
- Sem OOM → seguir etapa 3.

---

### 3. Validar requests/limits vs. consumo real

Comandos:

- `kubectl -n production get deploy chronos-api -o yaml | grep -A10 resources:`
- `kubectl -n production top pods -l app=chronos-api`
- `kubectl -n production exec -it <pod_name> -- sh -c "cat /sys/fs/cgroup/memory.max 2>/dev/null || cat /sys/fs/cgroup/memory/memory.limit_in_bytes"`

Verificação esperada: consumo real próximo ou acima do limits.memory configurado.

Próxima ação:

- Limits subdimensionados → preparar patch emergencial de memória (etapa 5c).
- Limits adequados e consumo anômalo → seguir etapa 4.

---

### 4. Correlacionar com deploy recente e dependências externas

Comandos:

- `argocd app history chronos-api | head -10`
- `kubectl -n production logs deploy/chronos-api --since=20m | grep -iE "oom|timeout|ledger|reactor|connection pool|error"`
- `aws sqs get-queue-attributes --queue-url <reactor_queue_url> --attribute-names ApproximateNumberOfMessages ApproximateAgeOfOldestMessage`

Verificação esperada: deploy nas últimas 2h coincidindo com início do alerta, backlog do Reactor acima do normal ou timeouts de Ledger nos logs.

Próxima ação:

- Regressão pós-deploy confirmada → rollback via Argo CD (etapa 5b).
- Backlog do Reactor crescendo → escalar dependência e aplicar mitigação da etapa 5.
- Sem causa externa → revisar métricas no Grafana (dashboard Chronos API / Memory).

---

### 5. Mitigação imediata (baixo risco)

5a — Aumentar réplicas (se houver capacidade de cluster):

- `kubectl -n production scale deploy chronos-api --replicas=10`
- `kubectl -n production rollout status deploy/chronos-api`

5b — Rollback via Argo CD (se regressão confirmada):

- `argocd app rollback chronos-api`
- `argocd app wait chronos-api --health`

5c — Ajuste emergencial de memory limit:

- `kubectl -n production patch deploy chronos-api -p '{"spec":{"template":{"spec":{"containers":[{"name":"api","resources":{"limits":{"memory":"1536Mi"},"requests":{"memory":"768Mi"}}}]}}}}'`

Verificação esperada: memória média dos pods caindo abaixo de 80% em até 10 min.

Próxima ação:

- Estabilizou → manter monitoramento por 20 min antes de encerrar.
- Não estabilizou → escalar imediatamente (seção abaixo).

---

### Escalação para @chronos-core

Escalar se qualquer critério abaixo for verdadeiro:

- Pressão de memória persiste >15 min em horário comercial ou >30 min fora.
- OOM recorrente após mitigação aplicada.
- HPA no teto máximo por >10 min sem queda de consumo.
- Backlog do Reactor crescendo continuamente.
- Rollback falhou ou o deploy anterior também apresentou o problema.

Mensagem mínima no #oncall-chronos: @chronos-core — Alerta de memória em chronos-api/production. Início: <timestamp>. HPA em <N/12>. OOM: <sim/não>. Deploy recente: <versão>. Ações: <etapas executadas>. Estado atual: <descritivo>.

---

### Encerramento do incidente

Encerrar apenas se todos os critérios forem verdadeiros por 20 min consecutivos:

- Memória média dos pods <75%.
- Nenhum OOM ou restart novo.
- Taxa de erro da aplicação normalizada.
- Backlog do Reactor em queda sustentada.
- HPA abaixo do teto máximo.

Checklist pós-incidente:

- Registrar timeline completa e hipótese de causa raiz.
- Abrir action items com owner e prazo (ex: tuning de memory requests/limits, revisão de Lifecycle Policy do Argo CD).
- Revisar thresholds do alerta (85% pode ser tardio; avaliar 75% como warning).
- Validar se memory requests refletem baseline real de consumo da versão atual.
- Atualizar este runbook se qualquer passo falhou ou ficou ambíguo.
```

## Justificativa

Role define a profundidade técnica e o tipo de decisão esperado de um SRE de plantão. Input concentra os dados operacionais necessários para diagnóstico sem depender de contexto externo. Steps transforma o pedido em procedimento reprodutível com sequência objetiva de execução. Expectation fixa critérios de completude e remove ambiguidade na forma final do runbook.
