# Questão 05 - Modernizar deployment legado

## Prompt

```text
# Before
Tenho um Deployment Kubernetes legado em produção com problemas críticos:
- imagem usando tag latest
- secrets hardcoded no manifest
- apenas 1 réplica
- sem requests/limits
- sem liveness/readiness probes
- sem securityContext não-root
- sem práticas modernas de rollout

Manifest atual:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: chronos-api
  template:
    metadata:
      labels:
        app: chronos-api
    spec:
      containers:
      - name: api
        image: chronos-api:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_PASSWORD
          value: "P@ssw0rd2023!"
        - name: JWT_SECRET
          value: "hvt-jwt-prod-secret"

# After
Quero um Deployment moderno e pronto para produção, com:
- alta disponibilidade
- imagem versionada (sem latest)
- secrets externos via Secret/secretKeyRef
- resource requests e limits
- liveness e readiness probes
- securityContext não-root
- estratégia de rollout segura
- anotações e labels consistentes para operação

# Bridge
Transforme o manifest atual no manifest modernizado completo (YAML válido), mantendo o mesmo nome de app e namespace. Entregue apenas o YAML final e, em seguida, uma lista curta (5 a 8 bullets) com as melhorias aplicadas.
```

## Modelo

OpenAI GPT-5.3-Codex. Escolhi este modelo pela capacidade de raciocinar sobre dívidas técnicas em manifests Kubernetes e aplicar, em conjunto, padrões de segurança, disponibilidade e performance.

## Output

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
  labels:
    app: chronos-api
    app.kubernetes.io/name: chronos-api
    app.kubernetes.io/part-of: chronos
    app.kubernetes.io/component: backend
  annotations:
    owner.team: platform-sre
    environment: production
spec:
  replicas: 3
  minReadySeconds: 15
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: chronos-api
  template:
    metadata:
      labels:
        app: chronos-api
        app.kubernetes.io/name: chronos-api
        app.kubernetes.io/part-of: chronos
        app.kubernetes.io/component: backend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      serviceAccountName: chronos-api
      terminationGracePeriodSeconds: 30
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
        fsGroup: 20001
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - chronos-api
                topologyKey: kubernetes.io/hostname
      containers:
        - name: api
          image: chronos-api:v2.49.3
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
              name: http
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: chronos-api-secrets
                  key: DB_PASSWORD
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: chronos-api-secrets
                  key: JWT_SECRET
          resources:
            requests:
              cpu: "250m"
              memory: "384Mi"
            limits:
              cpu: "750m"
              memory: "768Mi"
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 10001
            runAsGroup: 10001
            capabilities:
              drop: ["ALL"]
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            initialDelaySeconds: 8
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            initialDelaySeconds: 20
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /health/startup
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 12
            timeoutSeconds: 2
          terminationMessagePolicy: FallbackToLogsOnError
```

- Alta disponibilidade com 3 réplicas, rolling update sem indisponibilidade e anti-affinity para reduzir concentração no mesmo nó.
- Imagem versionada (`chronos-api:v2.49.3`), removendo uso de `latest`.
- Secrets externos via `secretKeyRef`, sem credenciais hardcoded no manifest.
- Requests e limits definidos para controle de capacidade e estabilidade do cluster.
- Probes de `startup`, `readiness` e `liveness` para ciclo de vida completo da aplicação.
- `securityContext` com execução não-root, `seccomp` padrão, sem privilege escalation e com capabilities mínimas.
- Labels e anotações operacionais consistentes para observabilidade, ownership e gestão em produção.

## Justificativa

Before deixa explícito o estado inseguro e limitado do deployment atual. After define o estado-alvo com critérios claros de produção moderna. Bridge pede a transformação completa do artefato atual para o desejado, sem ambiguidade de entrega. O B-A-B foi ideal porque o problema é uma migração de estado legado para estado padrão.
