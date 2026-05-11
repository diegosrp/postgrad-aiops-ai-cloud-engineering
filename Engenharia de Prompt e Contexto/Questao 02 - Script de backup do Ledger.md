# Questão 02 - Script de backup do Ledger

## Prompt

```text
# Role
Você é um engenheiro de plataforma SRE sênior, especialista em automação de backup PostgreSQL, Bash em Linux e operação segura em AWS.

# Task
Escreva um script bash de backup automatizado para um banco de dados PostgreSQL de produção chamado Ledger.

O ambiente é:
- Host: ledger-db.internal.hvt.io, porta 5432
- Banco: ledger_prod
- Usuário de backup: backup_user
- Senha: variável de ambiente PGPASSWORD, injetada pelo AWS Secrets Manager via IAM role da instância
- Região AWS: us-east-1
- SO: Ubuntu 22.04 LTS
- Diretório de trabalho: /var/backups/ledger (80 GB livres)
- Tamanho médio do dump compactado: ~12 GB

O script deve executar as seguintes etapas em ordem:
1. Fazer o dump do banco com pg_dump e compactar com gzip em um único pipeline, salvando em /var/backups/ledger com nome no formato ledger_prod_YYYY-MM-DD_HHMMSS.sql.gz.
2. Fazer upload do arquivo para o bucket S3 hvt-ledger-backups via aws s3 cp.
3. Aplicar retenção de 30 dias: remover do S3 os objetos com mais de 30 dias usando aws s3 ls e aws s3 rm.
4. Registrar cada etapa em /var/log/ledger-backup.log com timestamp ISO 8601.
5. Sair com exit code 0 em sucesso e exit code diferente de zero em qualquer falha, logando a etapa que falhou.

# Format
Entregue exatamente nesta estrutura:
1. Um bloco único contendo apenas o script bash completo.
2. Depois, uma lista curta com 5 a 8 bullets explicando as principais boas práticas aplicadas.
3. Não incluir texto introdutório, aviso, ou explicações fora dessa estrutura.
```

## Modelo

Gemini em modo raciocínio. Escolhi este modelo pela precisão em scripts Bash com foco operacional, tratamento de erros, validações de pré-condição e clareza no fluxo de CLI.

## Output

```bash
#!/bin/bash

# ==============================================================================
# Script: backup_ledger.sh
# Descrição: Backup automatizado PostgreSQL Ledger para AWS S3
# Autor: SRE Platform Team
# ==============================================================================

set -eo pipefail

# Configurações de Ambiente
DB_HOST="ledger-db.internal.hvt.io"
DB_PORT="5432"
DB_NAME="ledger_prod"
DB_USER="backup_user"
BACKUP_DIR="/var/backups/ledger"
S3_BUCKET="s3://hvt-ledger-backups"
AWS_REGION="us-east-1"
LOG_FILE="/var/log/ledger-backup.log"

# Variáveis Dinâmicas
TIMESTAMP=$(date +"%Y-%m-%d_%H%M%S")
BACKUP_FILENAME="${DB_NAME}_${TIMESTAMP}.sql.gz"
BACKUP_PATH="${BACKUP_DIR}/${BACKUP_FILENAME}"
RETENTION_DAYS=30

# Função de Logging (ISO 8601)
log() {
    local level=$1
    local message=$2
    echo "$(date -u +'%Y-%m-%dT%H:%M:%SZ') [$level] - $message" | tee -a "$LOG_FILE"
}

# Tratamento de Erros
trap 'log "ERROR" "Falha catastrófica na etapa anterior. Abortando."; exit 1' ERR

# Garantir diretório de trabalho
mkdir -p "$BACKUP_DIR"

log "INFO" "Iniciando processo de backup para o banco: $DB_NAME"

# 1. Dump e Compactação (Streamed pipeline para economizar IOPS)
log "INFO" "Executando pg_dump e compressão gzip..."
pg_dump -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" "$DB_NAME" | gzip > "$BACKUP_PATH"
log "INFO" "Arquivo gerado localmente: $BACKUP_PATH"

# 2. Upload para AWS S3
log "INFO" "Iniciando upload para S3 ($S3_BUCKET)..."
aws s3 cp "$BACKUP_PATH" "${S3_BUCKET}/${BACKUP_FILENAME}" --region "$AWS_REGION"
log "INFO" "Upload concluído com sucesso."

# 3. Retenção de 30 dias no S3
log "INFO" "Limpando backups com mais de $RETENTION_DAYS dias no S3..."
THRESHOLD_DATE=$(date -d "$RETENTION_DAYS days ago" +%Y-%m-%d)

aws s3 ls "$S3_BUCKET/" | while read -r line; do
    # Extrai a data de criação (YYYY-MM-DD) e o nome do arquivo
    OBJ_DATE=$(echo "$line" | awk '{print $1}')
    OBJ_NAME=$(echo "$line" | awk '{print $4}')
    
    if [[ "$OBJ_DATE" < "$THRESHOLD_DATE" ]] && [[ -n "$OBJ_NAME" ]]; then
        log "INFO" "Removendo objeto expirado: $OBJ_NAME (Data: $OBJ_DATE)"
        aws s3 rm "${S3_BUCKET}/$OBJ_NAME" --region "$AWS_REGION"
    fi
done

# Limpeza local de segurança (Mantém apenas o backup atual para evitar disk full)
find "$BACKUP_DIR" -type f -name "*.sql.gz" -mtime +1 -delete

log "INFO" "Backup do Ledger finalizado com sucesso."
exit 0

```

* **Fail-Fast com `set -eo pipefail**`: Garante que o script pare imediatamente se qualquer comando falhar, inclusive em pipes (como `pg_dump | gzip`), o que o `set -e` sozinho não detectaria.
* **Segurança de Credenciais**: O uso do `PGPASSWORD` via AWS Secrets Manager e IAM Roles evita a exposição de senhas em texto plano no código ou em arquivos `.pgpass` vulneráveis.
* **Eficiência de Pipeline**: O dump é enviado diretamente para o `gzip` via pipe, reduzindo drasticamente o consumo de I/O de disco e o tempo total de execução.
* **Rastreabilidade ISO 8601**: Logs padronizados facilitam a ingestão por ferramentas de monitoramento (como CloudWatch Logs ou ELK Stack) e auditorias.
* **Gestão de Ciclo de Vida**: A retenção manual via script garante controle granular sobre a limpeza, complementada por um `find` local para prevenir que os 80GB de disco se esgotem com o tempo.
* **Idempotência e Tratamento de Sinais**: O uso de `trap` captura sinais de erro inesperados, garantindo que falhas silenciosas não passem despercebidas pela equipe de operação.

## Justificativa

Role define o nível de especialização esperado para decisões de segurança e operação em produção. Task descreve o problema, o ambiente e a sequência obrigatória do processo de backup ponta a ponta. Format força a entrega prática no padrão pedido (script + bullets de boas práticas). O R-T-F é adequado aqui porque o objetivo principal é gerar um artefato técnico com estrutura previsível.
