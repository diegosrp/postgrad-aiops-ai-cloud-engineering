# Questão 08 - Postmortem técnico de incidente em produção

## Prompt

```text
# Role
Você é Staff SRE liderando resposta a incidente crítico em produção e precisa produzir um postmortem técnico decisório em até 20 minutos.

# Input
Deploy anterior (2026-04-23 18:42 UTC):
- chronos-api v2.47.0 -> v2.48.0
- novo endpoint POST /v2/transactions/batch
- refatoração do cliente Ledger (pool em nova lib)
- psycopg 3.1.18 -> 3.2.0
- timeout Ledger reduzido de 5s para 2s

Métricas 30 min:
- p99 latência: 420 -> 8100 ms
- req_rate: 1200 -> 2650 req/s
- err_rate: 0,2% -> 11,7%

Logs:
- connection pool exhausted (max=20, active=20, waiting=147)
- query timeout after 2000ms
- POST /v2/transactions/batch failed: context deadline exceeded
- circuit-breaker OPEN (87%)

Reactor:
- 50.127 mensagens acumuladas, +800/min
- consumer lag 18 min e aumentando

Cluster:
- Chronos 12/12 pods (HPA no máximo)
- CPU pods 62%, memória 71%
- Ledger connections 240/250

# Steps
1. Produza um postmortem técnico curto com: resumo do impacto, timeline, hipótese raiz, evidências e decisão recomendada.
2. Escolha explicitamente entre:
   - rollback do deploy v2.48.0
   - scaling emergencial (RDS limits + pool de conexões)
3. Justifique a decisão com base nos dados, incluindo risco da opção não escolhida.
4. Traga plano de ação em duas fases:
   - imediata (0 a 30 min)
   - estabilização (30 a 180 min)
5. Inclua critérios objetivos de sucesso para encerrar estado crítico.

# Expectation
Entregar uma análise acionável para tomada de decisão imediata, com conclusão inequívoca e plano executável pelo time de incidente.

# Format
- Título de decisão (Rollback ou Scaling).
- Seções: Impacto, Evidências, Causa Provável, Decisão, Plano Imediato, Plano de Estabilização, Critérios de Sucesso, Riscos remanescentes.
```

## Modelo

OpenAI GPT-5.4. Escolhi este modelo pela capacidade de correlacionar métricas, logs e mudanças de deploy sob pressão de tempo, estruturando uma decisão técnica clara, defensável e acionável.

## Output

```markdown
## Decisão: ROLLBACK do deploy v2.48.0

### Impacto
Incidente crítico em produção com degradação severa de latência e disponibilidade logo após o deploy de `chronos-api v2.48.0`. Em 30 minutos, o p99 subiu de 420 ms para 8100 ms, a taxa de erro foi de 0,2% para 11,7% e o backlog do Reactor alcançou 50.127 mensagens, crescendo a +800/min. O HPA já está em 12/12 pods, sem evidência de saturação de CPU ou memória, o que indica que ampliar compute na aplicação não resolve a causa dominante.

### Evidências
- A linha do tempo aponta correlação direta com o deploy de `2026-04-23 18:42 UTC`, que introduziu três mudanças de alto risco ao mesmo tempo: novo endpoint `POST /v2/transactions/batch`, refatoração do cliente Ledger com novo pool e redução de timeout de 5s para 2s.
- O log `connection pool exhausted (max=20, active=20, waiting=147)` mostra contenção aguda no acesso ao Ledger, compatível com explosão de concorrência por requisição.
- O log `query timeout after 2000ms` coincide exatamente com a nova configuração de timeout, sugerindo que o sistema passou a falhar antes de absorver variações transitórias de latência do Ledger.
- `circuit-breaker OPEN (87%)` confirma degradação sistêmica da dependência e perda de capacidade efetiva do serviço.
- `Ledger connections 240/250` mostra que o banco já está próximo da exaustão global; escalar pool ou limite neste ponto aumenta pressão sobre a mesma dependência já saturada.
- CPU em 62% e memória em 71% com HPA no máximo reforçam que o gargalo principal não é compute no cluster, mas contenção de conexões e timeout no caminho para o Ledger.

### Causa provável
A hipótese mais provável é uma regressão introduzida no v2.48.0 pela combinação de novo endpoint batch, nova implementação de pool do cliente Ledger e timeout reduzido para 2 segundos. O efeito conjunto aumentou concorrência por conexão, elevou filas de espera no pool, antecipou timeouts de query e abriu o circuit breaker, retroalimentando erro, backlog e latência.

### Decisão
**Executar rollback imediato para v2.47.0.**

Esta é a opção com maior probabilidade de remover rapidamente a regressão introduzida no deploy mais recente. A alternativa de scaling emergencial de RDS limits e pool de conexões trata o sintoma, não a causa, e carrega risco elevado de ampliar a pressão sobre um Ledger já em `240/250` conexões. Em outras palavras: escalar agora pode prolongar a janela crítica, aumentar contenção no banco e mascarar temporariamente uma regressão funcional que continuará ativa.

### Plano imediato (0 a 30 min)
1. Executar rollback via Argo CD para `chronos-api v2.47.0`.
2. Congelar novos deploys e mudanças de configuração em Chronos até sair do estado crítico.
3. Monitorar em janelas de 5 minutos: `err_rate`, `p99`, backlog do Reactor, consumer lag e conexões do Ledger.
4. Se o endpoint batch continuar recebendo tráfego relevante durante a drenagem, aplicar limitação temporária ou desabilitação controlada na borda.
5. Comunicar status ao incidente commander com checkpoint objetivo: rollback concluído, tendência das métricas e risco residual do backlog.

### Plano de estabilização (30 a 180 min)
1. Confirmar hipótese raiz comparando comportamento do v2.47.0 contra o v2.48.0 em tráfego controlado.
2. Revisar a política de pool do cliente Ledger: tamanho, fila de espera, timeout e estratégia de retry.
3. Reavaliar o timeout de 2s; definir valor compatível com latência real observada e orçamento de erro.
4. Reintroduzir o endpoint batch apenas com feature flag, limite de concorrência e teste progressivo.
5. Criar guardrails permanentes: alerta de pool exhaustion, alerta antecipado de circuit breaker e orçamento de conexões do Ledger.

### Critérios de sucesso
- `p99 < 800 ms` por 20 minutos consecutivos.
- `err_rate < 1%` por 20 minutos consecutivos.
- Backlog do Reactor em queda sustentada até inverter a tendência de crescimento.
- Consumer lag voltando para menos de 5 minutos e reduzindo continuamente.
- Conexões do Ledger abaixo de 80% do limite por 20 minutos consecutivos.

### Riscos remanescentes
- O backlog acumulado pode sustentar latência elevada mesmo após o rollback, exigindo drenagem monitorada.
- Se houver mudança de esquema ou efeito colateral não reversível no deploy, parte da degradação pode persistir após voltar para `v2.47.0`.
- Um pico secundário pode ocorrer durante o reprocessamento da fila; por isso, conexões do Ledger e circuit breaker devem permanecer sob observação contínua.
```

## Justificativa

Escolhi R-I-S-E porque a situação pede resposta operacional sob pressão de tempo: Role estabelece o nível de decisão, Input concentra evidências brutas, Steps obriga sequência de análise/decisão e Expectation fecha critérios objetivos de saída.

Comparação com T-A-G: eu ganharia uma narrativa forte de meta (restaurar SLA rapidamente), porém perderia o detalhamento procedural por etapa, que é essencial para condução tática do incidente em tempo real.

Comparação com B-A-B: eu ganharia clareza do contraste estado atual vs estado desejado, útil para comunicar o quadro executivo, mas perderia precisão na condução minuto a minuto (diagnóstico, decisão e execução), que é o núcleo desta demanda.

Por isso, R-I-S-E oferece o melhor equilíbrio entre velocidade de decisão, rastreabilidade técnica e execução prática durante o incidente.
