# Holistic Health Bytes

Monitoramento holístico **self-hosted**: dashboard PWA, agentes leves e plugins — organizações, aplicações, clusters Kubernetes, fornecedores e alertas em um só lugar.

Este repositório é o canal **público de distribuição** (releases e documentação de instalação). Os binários ficam em [Releases](https://github.com/lucresce/holistic-health-bytes-releases/releases); a imagem Docker é publicada no GHCR.

**Última versão documentada neste README:** `v1.0.0-rc.43`  
Sempre prefira a tag do [latest release](https://github.com/lucresce/holistic-health-bytes-releases/releases/latest).

---

## Features

- **Dashboard PWA** — visão de saúde, alertas e ações sugeridas
- **Organizações e aplicações** — multi-tenant leve, apps e componentes
- **Composição** — grafo de dependências (app → componente → cluster/host/fornecedor) com criticidade
- **Planos de monitoração** — o que monitorar, quando alertar e quem notificar
- **Agentes + plugins** — instalação pelo dashboard; updates remotos quando o agente for compatível
- **Kubernetes** — nodes, pods, deployments, jobs/cronjobs; descoberta via runner do cluster
- **Fornecedores** — status page / HTTP / TCP (AWS, GCP, Azure, custom, etc.)
- **Alertas e operações** — severidade, canais, filas/tickets e incidentes (conforme edição)

---

## Instalação (servidor)

### Requisitos

- Docker Engine + Docker Compose
- PostgreSQL (o compose de referência sobe um Postgres; em produção use um gerenciado)
- Porta `9090` (ou reverse proxy HTTPS)
- `public_url` alcançável pelos agentes (URL pública do Holistic)

### Início rápido com Docker

```bash
export HHB_VERSION=v1.0.0-rc.43   # ou a tag do latest release

docker pull ghcr.io/lucresce/holistic-health-bytes:${HHB_VERSION}

mkdir -p data
cat > config.yaml <<EOF
listen: ":9090"
data_dir: "/app/data"
database:
  driver: "postgres"
  dsn: "postgres://holistic:change-me@postgres:5432/holistic?sslmode=disable"
master_key: "SUBSTITUA_COM_openssl_rand_hex_32"
public_url: "https://SEU_DOMINIO"
agent_bin: "/app/bin/agent"
github_repo: "lucresce/holistic-health-bytes-releases"
tls:
  cert: ""
  key: ""
EOF
```

Suba Postgres + app (exemplo mínimo):

```bash
docker network create hhb || true

docker run -d --name hhb-postgres --network hhb --network-alias postgres \
  -e POSTGRES_DB=holistic \
  -e POSTGRES_USER=holistic \
  -e POSTGRES_PASSWORD=change-me \
  postgres:16-alpine

docker run -d --name hhb --network hhb -p 9090:9090 \
  -v "$PWD/config.yaml:/app/config.yaml:ro" \
  -v "$PWD/data:/app/data" \
  ghcr.io/lucresce/holistic-health-bytes:${HHB_VERSION}
```

Abra a `public_url` e crie a conta **admin** no primeiro acesso.

> Em produção: use HTTPS no proxy, `master_key` estável, Postgres com senha forte/`sslmode=require`, e volume persistente em `/app/data` (UID `1000` — no Kubernetes use `fsGroup: 1000`).

### Atualizar o servidor

```bash
export HHB_VERSION=v1.0.0-rc.43   # nova tag
docker pull ghcr.io/lucresce/holistic-health-bytes:${HHB_VERSION}
docker stop hhb && docker rm hhb
# rode de novo o mesmo docker run com a nova tag
```

Agentes e plugins buscam binários neste repositório (`github_repo`). Com o servidor atualizado, o dashboard indica updates e pode atualizar agentes compatíveis remotamente.

### Config importante

| Campo | Função |
|-------|--------|
| `public_url` | URL que os agentes usam para falar com o servidor |
| `master_key` | Chave estável (não use placeholder em produção) |
| `github_repo` | Deve ser `lucresce/holistic-health-bytes-releases` |
| `database.dsn` / `HHB_DATABASE_URL` | PostgreSQL |

Healthchecks: `GET /healthz` (liveness), `GET /readyz` (readiness), `GET /api/v1/setup/status` (smoke).

---

## Assets

### Imagem Docker (multi-arch)

| Tag | Uso |
|-----|-----|
| `ghcr.io/lucresce/holistic-health-bytes:v1.0.0-rc.43` | versão pinada |
| `ghcr.io/lucresce/holistic-health-bytes:latest` | última publicada |

Plataformas: `linux/amd64`, `linux/arm64` e agente `windows/amd64`.

Layout na imagem:

| Path | Uso |
|------|-----|
| `/app/server` | servidor (`ENTRYPOINT`) |
| `/app/config.yaml` | config padrão (sobrescreva com mount) |
| `/app/data` | dados locais / cache de releases |
| `/app/bin/agent` | agente embutido |
| `/app/bin/plugins/*` | plugins embutidos |

```bash
docker pull ghcr.io/lucresce/holistic-health-bytes:v1.0.0-rc.43
```

### Binários no GitHub Release

Publicados em cada tag neste repositório:

| Asset | Arch |
|-------|------|
| `hhb-server-linux-{amd64,arm64}` | servidor bare-metal |
| `hhb-agent-linux-{amd64,arm64}` | agente Linux |
| `hhb-agent-windows-amd64` | agente Windows (sem self-update; reinstale via script) |
| `hhb-plugin-*-linux-{amd64,arm64}` | plugins (`host`, `updates`, `tcpcheck`, `k8s`, `logwatch`, `memory_analysis`, `httpcheck`, `process`, `systemd`, `statuspage`, …) |
| `hhb-plugin-*-windows-amd64` | subset compatível (sem `systemd`/`updates`) |

Lista e download: [Releases](https://github.com/lucresce/holistic-health-bytes-releases/releases).

O servidor Holistic usa a API deste repo para servir `/api/v1/agent/download` e `/api/v1/plugins/.../download` aos hosts.

---

## Cadastro de hosts

1. No dashboard: **+ Adicionar** servidor  
2. **Instalar manualmente** (recomendado): execute o script gerado no host  
   - Linux: script bash + systemd  
   - Windows: `install-script?os=windows` (PowerShell + serviço) — ver docs do produto  
3. Aguarde status **online**

O script baixa agente/plugins a partir do seu servidor; o servidor obtém os binários deste Release quando necessário.

---

## Links

- [Releases / changelog](https://github.com/lucresce/holistic-health-bytes-releases/releases)
- Imagem: `ghcr.io/lucresce/holistic-health-bytes`

O código-fonte do produto não é publicado neste repositório — apenas artefatos e documentação de instalação.
