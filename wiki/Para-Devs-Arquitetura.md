# Arquitetura

Visão técnica do ecossistema **Check-in Industrial**. Use esta página como porta de entrada — para profundidade, vá direto ao `CLAUDE.md` do repositório específico.

## Topologia

```
                            ┌──────────────────────────┐
                            │  Browser (cidadão/admin) │
                            └────────────┬─────────────┘
                                         │ HTTPS
                                         ▼
                            ┌──────────────────────────┐
                            │  checkin-industrial-     │
                            │  painel  (React 19)      │
                            └────────────┬─────────────┘
                                         │ /api/* + X-Api-Key
                                         ▼
                            ┌──────────────────────────┐
                            │  checkin-industrial-     │
                            │  api  (.NET 10)          │
                            └────────────┬─────────────┘
                                         │
                                         ▼
                            ┌──────────────────────────┐
                            │  PostgreSQL 16           │
                            └──────────────────────────┘
```

## Repositórios

| Repo | Stack | Papel |
|---|---|---|
| [`checkin-industrial-api`](https://github.com/checkin-industrial/checkin-industrial-api) | .NET 10 + EF Core + Postgres | Backend REST com 5 features VSA (Empresas, Pontos Institucionais, Telefones Úteis, Analytics, Geocoding) |
| [`checkin-industrial-painel`](https://github.com/checkin-industrial/checkin-industrial-painel) | React 19 + Vite 7 + TanStack Query | Widget público + área administrativa |
| [`checkin-industrial-tests-e2e`](https://github.com/checkin-industrial/checkin-industrial-tests-e2e) | Robot Framework 7.2 + Allure | Suíte E2E rodando contra Docker Compose real |
| [`checkin-industrial-docs`](https://github.com/checkin-industrial/checkin-industrial-docs) | Markdown + GitHub Wiki | Este repositório (Wiki + apresentação comercial) |
| [`checkin-industrial-infra`](https://github.com/checkin-industrial/checkin-industrial-infra) | (a definir) | Placeholder para Terraform/k8s/etc. quando a tecnologia de deploy for escolhida |

## Decisões arquiteturais relevantes

### 1. Vertical Slice Architecture (VSA) na API

Cada feature é uma **pasta auto-contida** em `src/Features/<Feature>/` com entidade, services, queries, DTOs e arquivos de endpoint (um endpoint = um arquivo). Não há separação Domain/Application/Infrastructure.

Por quê: o sistema é mantido majoritariamente por agentes de IA, e VSA reduz drasticamente o número de arquivos que o agente precisa abrir para entender ou modificar uma feature. Detalhes em [`src/Features/CLAUDE.md`](https://github.com/checkin-industrial/checkin-industrial-api/blob/main/src/Features/CLAUDE.md) da API.

### 2. Soft delete consistente

Todas as 3 features de CRUD (Empresas, Pontos Institucionais, Telefones Úteis) implementam **soft delete** via campo `Ativo` (bool):

- `DELETE /api/<recurso>/{id}` → seta `Ativo=false`. Idempotente.
- Painel admin tem dropdown "Ativas / Inativas / Todas" + botão "Reativar" para registros inativos.
- Widget público filtra por `?ativo=true` automaticamente — soft-deletados não aparecem para o cidadão.

### 3. Auth por API Key (sem JWT, sem OAuth)

Endpoints de escrita exigem header `X-Api-Key`. A chave é fixa por instância, distribuída fora-de-banda para administradores autorizados. No painel, é digitada via modal de login que persiste em `sessionStorage` (some ao fechar a aba — escolha de segurança).

Validação no painel: chama `DELETE /api/empresas/{guid-impossivel}` — 401 indica chave inválida, 404 (ou qualquer outro) indica chave válida.

### 4. TanStack Query no painel

Todas as features de leitura usam `useQuery` com a convenção `queryKey: [FEATURE_KEY, ...variantes]`. Mutações invalidam pela `queryKey` raiz para refrescar todas as variantes.

Configuração global: `staleTime: 5min`, `retry: 1`, `refetchOnWindowFocus: false` (widget — não queremos refetch em foco).

### 5. Auth admin no painel

Navegação para qualquer aba de Gestão é **interceptada** quando o usuário não está autenticado — abre o modal de login antes de navegar. Logout volta para a tab pública padrão (Mapa).

## Convenções operacionais

### Branch protection

Todos os repos exigem para merge em `main`:
- Aprovação de Code Owner
- Resolução de todas as conversas do PR
- CI verde (pipelines required)

Bypass: o owner da org está na lista de exceções para casos de unblock urgente.

### Publish da imagem Docker

A cada push em `main` do `checkin-industrial-api`, o workflow `publish-docker.yml` builda multi-arch (amd64+arm64) e publica em `checkinindustrial/checkin-industrial-api:latest` (mais `:main`, `:sha-<short>` e, em tags `v*`, semver).

A suíte E2E **puxa essa imagem** em CI — não builda do código. Isso garante que a suíte testa exatamente o que vai pra produção.

### Documentação

Padrão `CLAUDE.md` em cada repo + sub-pastas relevantes. **Este Wiki** é a porta de entrada de alto nível; conteúdo profundo fica nos `CLAUDE.md` (single source of truth — sem duplicação).
