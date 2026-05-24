# Como Rodar Localmente

Setup completo do ecossistema na sua máquina. Tempo estimado: **10 minutos** após o primeiro build.

## Pré-requisitos

- Docker 24+ com Docker Compose v2
- .NET 10 SDK (opcional — só se for rodar a API fora do compose)
- Node 20+ (opcional — só se for rodar o painel fora do compose)
- Python 3.11+ (opcional — só para executar a suíte E2E)

## Layout esperado

Clone os repos como **pastas irmãs**:

```text
c:/git/checkin-industrial/
├── checkin-industrial-api/
├── checkin-industrial-painel/
├── checkin-industrial-tests-e2e/
├── checkin-industrial-docs/
└── checkin-industrial-infra/  (placeholder)
```

Vários arquivos (docker-compose, scripts) usam paths relativos assumindo essa estrutura.

## Subir tudo via Docker Compose

A composição **dev local** roda API + Painel + Postgres:

```bash
cd checkin-industrial-tests-e2e/docker-compose
docker compose up -d --build
```

- API: <http://localhost:8080> (Swagger na raiz)
- Painel: <http://localhost:8081>
- Postgres: localhost:5432 (user `postgres`, pw `postgres`, db `turismoindustrial`)

Para subir só o ambiente da suíte E2E (Postgres + API, sem Painel), com env vars de teste:

```bash
docker compose -f docker-compose/docker-compose.e2e.yml up -d --wait --pull always
```

## Rodar a API standalone (dev iteration)

```bash
cd checkin-industrial-api/src
dotnet run
```

Requer um Postgres ouvindo em `localhost:5432`. O `appsettings.json` tem defaults dev (`Server=localhost;Password=postgres`).

Para mudar a connection string sem editar o appsettings:

```bash
export ConnectionStrings__DefaultConnectionTurismo="Server=...;..."
dotnet run
```

## Rodar o Painel standalone (dev iteration)

```bash
cd checkin-industrial-painel
npm install
VITE_DEV_PROXY_TARGET=http://localhost:8080 npm run dev
```

App em <http://localhost:5173>. O proxy do Vite redireciona `/api` e `/uploads` para a API local.

## Rodar a suíte E2E

```bash
cd checkin-industrial-tests-e2e
pip install -r requirements.txt
docker compose -f docker-compose/docker-compose.e2e.yml up -d --wait
robot --outputdir results tests/suites/
```

Report em `results/report.html`.

## API Key admin em desenvolvimento

A composição E2E define `Auth__ApiKey=e2e-api-key-checkin-industrial-2026`. Use essa chave no modal de login do Painel ao acessar a aba **Gestão**.

Em produção, gere uma chave nova: `openssl rand -hex 32` e configure via `Auth__ApiKey` no host.

## Limpar tudo

```bash
docker compose -f docker-compose/docker-compose.e2e.yml down -v
docker compose -f docker-compose/docker-compose.yml down -v
```

`-v` remove os volumes (banco zerado na próxima subida).

## Troubleshooting comum

| Sintoma | Diagnóstico |
|---|---|
| `docker compose up` trava em healthcheck da API | Migration do EF rodando no startup. Aguarde 30s no primeiro run. Confirme `docker compose logs api`. |
| Painel mostra `Failed to fetch /api/...` | API não está acessível. Confira `curl http://localhost:8080/health` retorna `Healthy`. |
| 401 em endpoints de escrita | Faltou o `X-Api-Key`. Faça login no Painel via aba Gestão. |
| 404 em GET após POST recente | Output cache. Por default TTL=60s. Em E2E setamos `OutputCache__ReadEndpointTtlSeconds=0`. |
| 429 Too Many Requests | Rate limit (60/min anônimo). Aumente `RateLimit__AnonymousPermitPerMinute`. |
