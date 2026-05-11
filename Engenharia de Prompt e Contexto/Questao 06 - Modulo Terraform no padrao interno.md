# Questão 06 - Módulo Terraform no padrão interno

## Prompt

```text
# Context
Você é engenheiro de plataforma responsável por entregar um módulo Terraform reutilizável para buckets S3 no padrão interno da empresa.

Padrões obrigatórios:
- Tags em todo recurso: Owner, CostCenter, Environment.
- Prefixo hvt- nos nomes de recursos.
- Bucket S3 com encryption habilitada (SSE-S3 mínimo), versioning ativo, block public access total e logging configurado.
- Variáveis em variables.tf com description e type obrigatórios.
- Entregar exemplo de uso.

Referência de estilo existente:
variable "environment" {
  description = "Nome do ambiente (dev, staging, production)"
  type        = string
}

locals {
  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }
}

resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
  tags = merge(local.common_tags, {
    Name = "hvt-vpc-${var.environment}"
  })
}

# Action
Gere os arquivos `main.tf`, `variables.tf`, `outputs.tf` e um `README.md` curto com exemplo de uso, seguindo exatamente o padrão acima.

# Result
O módulo deve estar pronto para reuso por múltiplos times, com naming e tags consistentes, segurança mínima obrigatória e exemplo copiável.

# Example
No exemplo de uso, mostre uma chamada de módulo em `module "s3_logs" { ... }` com owner, cost_center, environment, bucket_suffix e log_bucket_name preenchidos.
```

## Modelo

OpenAI GPT-5.4. Escolhi este modelo pela consistência em seguir padrões de IaC com várias restrições simultâneas, gerando múltiplos arquivos Terraform com coerência estrutural e precisão técnica.

## Output

```hcl
# main.tf
locals {
  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }

  bucket_name = "hvt-${var.bucket_suffix}-${var.environment}"
}

resource "aws_s3_bucket" "this" {
  bucket = local.bucket_name

  tags = merge(local.common_tags, {
    Name = local.bucket_name
  })
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket = aws_s3_bucket.this.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_logging" "this" {
  bucket = aws_s3_bucket.this.id

  target_bucket = var.log_bucket_name
  target_prefix = "${local.bucket_name}/"
}
```

```hcl
# variables.tf
variable "owner" {
  description = "Responsável pelo recurso"
  type        = string
}

variable "cost_center" {
  description = "Centro de custo para alocação financeira"
  type        = string
}

variable "environment" {
  description = "Nome do ambiente (dev, staging, production)"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "environment deve ser dev, staging ou production."
  }
}

variable "bucket_suffix" {
  description = "Sufixo único para composição do nome do bucket"
  type        = string

  validation {
    condition     = length(var.bucket_suffix) > 0
    error_message = "bucket_suffix deve ser informado."
  }
}

variable "log_bucket_name" {
  description = "Bucket central de logs para access logging do S3"
  type        = string
}
```

```hcl
# outputs.tf
output "bucket_name" {
  description = "Nome do bucket criado"
  value       = aws_s3_bucket.this.bucket
}

output "bucket_arn" {
  description = "ARN do bucket criado"
  value       = aws_s3_bucket.this.arn
}

output "bucket_id" {
  description = "ID do bucket criado"
  value       = aws_s3_bucket.this.id
}
```

```md
# README.md
## Módulo S3 seguro

Cria um bucket S3 com prefixo `hvt-`, tags padronizadas, criptografia SSE-S3, versionamento, block public access total e access logging habilitado.

## Exemplo de uso

    module "s3_logs" {
      source          = "../modules/s3-secure"
      owner           = "plataforma"
      cost_center     = "CC-1001"
      environment     = "production"
      bucket_suffix   = "app-artifacts"
      log_bucket_name = "hvt-central-logs-production"
    }
```

## Justificativa

Context reúne regras corporativas, restrições de segurança e padrão de nomenclatura. Action pede explicitamente os arquivos e o escopo da implementação do módulo. Result define critérios de pronto para produção e reuso entre times. Example ancora o estilo e o formato esperado, reduzindo ambiguidade na saída. O C-A-R-E foi adequado porque havia referência concreta a ser seguida com alta consistência.
