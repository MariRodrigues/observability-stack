# 📊 Observability Stack

Stack de observabilidade com **Loki + Grafana**, pensada pra rodar num Raspberry Pi (cartão SD, sem SSD) de forma enxuta, e servir de destino de logs pra vários projetos ao mesmo tempo.

## 🧩 Tecnologias

- [Loki](https://grafana.com/oss/loki/) — armazena os logs (storage em filesystem local, sem banco externo)
- [Grafana](https://grafana.com/) — visualização/consulta dos logs via LogQL
- [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) — expõe Grafana e Loki na internet sem abrir porta nenhuma no roteador
- [Cloudflare Access](https://developers.cloudflare.com/cloudflare-one/policies/access/) — autentica quem acessa, na borda da Cloudflare (antes de chegar no Pi)

## 🏗️ Arquitetura

Dois `docker-compose` separados, de propósito:

- **`docker-compose.yaml`** — o "core": só Loki e Grafana. Não publica nenhuma porta pro host — só é alcançável de dentro da rede Docker. É esse arquivo que você leva pra uma VPS no futuro, sem alterar nada.
- **`docker-compose.tunnel.yaml`** — o `cloudflared`, responsável por expor os dois serviços acima. É específico de "estou hospedado atrás de CGNAT/sem IP público" — na VPS, provavelmente nem vai ser usado (a VPS tem IP próprio; expor lá vira outra decisão, ex: reverse proxy com TLS).

Os dois sempre sobem juntos no Pi, como um projeto Compose só:

```bash
docker compose -f docker-compose.yaml -f docker-compose.tunnel.yaml up -d
```

O `cloudflared` alcança os outros dois pelo nome do serviço (`http://loki:3100`, `http://grafana:3000`) porque, ao rodar os dois arquivos juntos, o Compose os trata como um único projeto e todos os containers caem na mesma rede padrão — não precisa configurar rede manualmente, só **sempre subir os dois arquivos juntos**.

Pra rodar só o core (ex: numa VPS futura, sem o túnel):

```bash
docker compose up -d
```

## ⚙️ Configuração

1. Copie `.env.example` para `.env` e preencha:
   - `GRAFANA_ADMIN_USER` / `GRAFANA_ADMIN_PASSWORD` — login do Grafana
   - `TUNNEL_TOKEN` — token do túnel (veja abaixo como gerar)

2. Suba a stack (comando acima).

## ☁️ Configurando o Cloudflare Tunnel + Access

Esses passos são feitos no [painel do Cloudflare Zero Trust](https://one.dash.cloudflare.com/) (não tem como automatizar isso pelo repositório — é conta/domínio da Cloudflare):

### 1. Criar o túnel
`Networks` → `Tunnels` → `Create a tunnel` → tipo *Cloudflared* → dê um nome (ex: `observability-pi`). Na etapa "Install connector", escolha Docker e copie só o token (é o valor depois de `--token`) — isso vai no `TUNNEL_TOKEN` do `.env`.

### 2. Rotear os hostnames (Public Hostnames, na mesma tela do túnel)
Adicione duas rotas:

| Subdomain | Service |
|---|---|
| `grafana.seudominio.com` | `HTTP` → `grafana:3000` |
| `loki.seudominio.com` | `HTTP` → `loki:3100` |

### 3. Proteger o Grafana (acesso humano)
`Access` → `Applications` → `Add an application` → `Self-hosted`. Domínio: `grafana.seudominio.com`. Policy: `Allow` pro seu e-mail (login via código enviado por e-mail, ou Google/GitHub SSO se preferir configurar).

### 4. Proteger a Loki (acesso só de aplicações, sem login humano)
Outra Access Application, domínio `loki.seudominio.com`. A policy aqui deve exigir **Service Auth** em vez de login — isso faz a Access aceitar só requisições que vierem com um Service Token (nenhum humano consegue entrar pelo navegador).

### 5. Gerar um Service Token por aplicação
`Access` → `Service Auth` → `Service Tokens` → `Create Service Token`. Cada projeto que vai mandar log ganha o seu (dá pra revogar um sem afetar os outros). A Cloudflare gera um `Client ID` e um `Client Secret` — a aplicação precisa mandar esses dois valores nos headers `CF-Access-Client-Id` e `CF-Access-Client-Secret` em toda requisição pra `loki.seudominio.com`.

> Nota: o sink `Serilog.Sinks.Grafana.Loki` não manda headers customizados por padrão — pra usar Service Token, a aplicação vai precisar de uma pequena implementação própria de `LokiHttpClient` adicionando esses dois headers. Isso é configuração do lado de cada projeto que loga, não desse repositório.

## 🔒 Por que a Loki não tem autenticação própria (`auth_enabled: false`)

A Loki não tem sistema de login embutido — quem autentica é sempre uma camada na frente dela (aqui, a Cloudflare Access). Como nenhuma porta é publicada pro host e o único caminho de entrada é via túnel + Access, isso é suficiente: ninguém alcança a Loki sem passar pelo Service Token antes.

## 📦 Retenção e uso do cartão SD

`loki-config.yaml` mantém `retention_period: 7d` — os logs mais antigos que isso são apagados automaticamente pelo compactor. Isso existe justamente pra não deixar o `loki-data/` crescer indefinidamente e desgastar o cartão SD. Ajuste esse valor se precisar guardar log por mais tempo (com o trade-off de mais escrita/espaço).

## 🚚 Migrando para uma VPS

Como o storage da Loki é filesystem local (sem banco externo) e as imagens (`grafana/loki`, `grafana/grafana`) são multi-arquitetura, a migração é só copiar dado:

1. No Pi: `docker compose down` (ou pelo menos parar os containers antes de copiar, pra não copiar arquivo sendo escrito)
2. Copiar as pastas `loki-data/` e `grafana-data/` inteiras pro novo host
3. Levar `docker-compose.yaml`, `loki-config.yaml` e `.env` (recriando os secrets)
4. Na VPS: `docker compose up -d` — mesmo arquivo, nenhuma alteração necessária
5. `docker-compose.tunnel.yaml` fica pra trás (a VPS tem IP próprio); resolver exposição por outro meio (reverse proxy com TLS, ou até manter o Cloudflare Tunnel se quiser continuar escondendo o IP de origem)
