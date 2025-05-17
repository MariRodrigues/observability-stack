# 📊 Observability Stack

Configuração de uma stack de observabilidade com **Loki + Grafana** usando Docker Compose.

## 🧩 Tecnologias Utilizadas

- [Grafana](https://grafana.com/) — Visualização de métricas e logs
- [Loki](https://grafana.com/oss/loki/) — Agregador de logs leve e eficiente
- [Docker Compose](https://docs.docker.com/compose/) — Orquestração dos serviços

## 🚀 Como rodar o projeto

1. Clone o repositório e navegue até a pasta do projeto:

    ```bash
    git clone <url-do-repositorio>
    cd observability-stack
    ```

2. Copie o arquivo `.env.example` para `.env` e ajuste as variáveis de ambiente conforme necessário:

    ```bash
    cp .env.example .env
    ```

3. Suba os containers em modo destacado (background):

    ```bash
    docker-compose up -d
    ```

4. Acesse o Grafana no navegador:

    ```
    http://localhost:3000
    ```

5. Faça login com as credenciais padrão (ou as definidas no `.env`):

    - Usuário: `admin`
    - Senha: `admin`

6. Configure o Loki como fonte de dados no Grafana (geralmente já configurado via `docker-compose`):

    - Vá em **Configuration > Data Sources**
    - Confirme que a URL do Loki está apontando para `http://loki:3100`

7. Explore os logs no Grafana:

    - Vá para a aba **Explore**
    - Selecione a fonte de dados **Loki**
    - Faça consultas para visualizar seus logs agregados
