# Questão 01 - Dockerfile para o Lift

## Prompt

```text
# Role
Você é um engenheiro sênior de DevOps e plataformas, especialista em Docker, Python/Flask, segurança de containers e boas práticas para workloads em Kubernetes.

# Task
Crie um Dockerfile de produção para uma API Python/Flask chamada Lift com estas características:
- Estrutura do projeto:
  - app.py
  - requirements.txt
  - lib/auth.py
  - lib/storage.py
  - tests/test_app.py

- A aplicação expõe a porta 8080.

- As dependências estão em requirements.txt com:
  - Flask==3.0.0
  - gunicorn==21.2.0
  - requests==2.31.0
  - python-dotenv==1.0.0
  - psycopg2-binary==2.9.9

- Em produção, o serviço sobe com:
  - gunicorn --bind 0.0.0.0:8080 --workers 4 app:app

- Existem duas variáveis de ambiente obrigatórias em runtime:
  - DATABASE_URL
  - API_KEY

Siga boas práticas de criação de imagem Docker, incluindo:
- imagem base enxuta e apropriada para produção
- configuração para logs sem buffering
- uso de diretório de trabalho
- cópia otimizada para aproveitar cache de camadas
- instalação sem arquivos desnecessários
- execução com usuário não-root
- exposição explícita da porta 8080
- comando final adequado para produção
- não embutir segredos no Dockerfile
- comentário curto apenas quando realmente útil

# Format
Entregue exatamente nesta estrutura:
1. Um bloco único contendo apenas o Dockerfile completo.
2. Depois, uma lista curta com 5 a 8 bullets explicando as principais boas práticas aplicadas.
3. Não incluir texto introdutório, aviso, ou explicações fora dessa estrutura.
```

## Modelo

OpenAI GPT-5.4. Escolhi este modelo pela consistência em tarefas de engenharia de software com múltiplas restrições técnicas, aplicando boas práticas de segurança, otimização de imagem e formato de saída com baixo retrabalho.

## Output

```dockerfile
FROM python:3.12-slim AS runtime

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /app

RUN groupadd --system app && \
    useradd --system --gid app --create-home --home-dir /home/app app

COPY requirements.txt ./

RUN pip install --no-cache-dir -r requirements.txt

COPY app.py ./
COPY lib ./lib

RUN chown -R app:app /app

USER app

EXPOSE 8080

CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "4", "app:app"]
```

- Usa uma imagem base enxuta com `python:3.12-slim`, reduzindo tamanho final e superfície de ataque.
- Define `PYTHONUNBUFFERED=1` para garantir logs imediatos em stdout/stderr, adequados para containers e Kubernetes.
- Copia `requirements.txt` antes do código da aplicação para maximizar reaproveitamento de cache nas camadas de build.
- Instala dependências com `--no-cache-dir`, evitando arquivos temporários desnecessários na imagem final.
- Define `WORKDIR /app` para padronizar o contexto de execução e simplificar manutenção.
- Cria e usa um usuário não-root (`app`), aplicando princípio de menor privilégio em runtime.
- Expõe explicitamente a porta 8080, consistente com o bind configurado no `gunicorn`.
- Mantém segredos fora da imagem; `DATABASE_URL` e `API_KEY` devem ser injetadas apenas em runtime.

## Justificativa

Role aparece logo no início ao definir a persona técnica (DevOps sênior com foco em Docker/Kubernetes). Task concentra os requisitos de build e runtime (dependências, porta, comando, variáveis obrigatórias e hardening). Format força uma saída operacional em dois blocos objetivos (Dockerfile e bullets). O R-T-F funcionou bem porque a tarefa é direta e o formato esperado é determinístico.
