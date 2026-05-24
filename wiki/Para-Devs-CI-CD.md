# CI/CD

Workflows GitHub Actions de cada repo + branch protection + publish pipelines.

## Visão geral

| Repo | Workflow | Trigger | O que faz |
|---|---|---|---|
| api | `ci.yml` | push em `main`, PR em `main` | `dotnet restore` + `build` + `test` |
| api | `publish-docker.yml` | push em `main`, tags `v*.*.*`, manual | Build multi-arch (amd64+arm64) + push pra Docker Hub |
| painel | `ci.yml` | push em `main`, PR em `main` | `lint` + `typecheck` + `test` + `build` + `audit` |
| tests-e2e | `e2e-tests.yml` | push em main/feature/refactor, PR em `main`, manual | Sobe stack + roda Robot + publica Allure no GitHub Pages |
| docs | `publish-wiki.yml` | push em `main` na pasta `wiki/` | Sincroniza `wiki/*.md` pro Wiki nativo do GitHub |

## Branch protection (todos os repos)

- Approving review por **Code Owner** obrigatório
- Resolução de **todas as conversas** do PR obrigatória
- **Required status checks**: as pipelines do repo precisam estar verdes
- **Last push approval**: aprovação fica inválida após novo push
- Bypass: owner da org está em `bypass_pull_request_allowances` para casos de unblock urgente

CODEOWNERS aponta para a team `@checkin-industrial/maintainers`.

## Publish da imagem Docker (API)

Workflow `publish-docker.yml`:

1. Trigger: push em `main` ou tag `v*.*.*`.
2. Login no Docker Hub via secret `DOCKERHUB_TOKEN` (access token, **nunca** a senha de login) + variable `DOCKERHUB_USERNAME`.
3. `docker/metadata-action` gera tags inteligentes:
   - Push em `main` → `:latest`, `:main`, `:sha-<short>`
   - Push em tag `v1.2.3` → `:latest`, `:v1.2.3`, `:1.2`, `:1`, `:sha-<short>`
4. `docker/build-push-action` builda multi-arch (amd64+arm64) com cache GHA + push pra Docker Hub.

Repositório no Docker Hub: <https://hub.docker.com/r/checkinindustrial/checkin-industrial-api>

## Suíte E2E + Allure no GitHub Pages

Workflow `e2e-tests.yml`:

1. Roda em push (branches main/feature/refactor), PR contra main, ou `workflow_dispatch`.
2. Puxa `checkinindustrial/checkin-industrial-api:${API_IMAGE_TAG:-latest}` do Docker Hub (sem build local — testa o que vai pra produção).
3. Sobe Postgres + API via Docker Compose, aguarda `--wait` (healthchecks).
4. Roda Robot com listener Allure (`robot --listener allure_robotframework:allure-results`).
5. Gera report Allure via CLI direta (Java 21 + allure 2.30.0).
6. Restaura history da branch `gh-pages` (preserva trend cross-runs).
7. Em push para `main`: publica em `gh-pages` via `peaceiris/actions-gh-pages`.

URL do report: <https://checkin-industrial.github.io/checkin-industrial-tests-e2e/> (requer Pages ativado).

Robot tem `continue-on-error: true` para garantir que o Allure seja gerado mesmo em falha; o job é marcado como `failure` ao final via step explícito que lê `steps.robot.outcome`.

## Publish do Wiki (Docs)

Este repositório (`checkin-industrial-docs`) tem workflow `publish-wiki.yml` que **sincroniza** a pasta `wiki/*.md` pro repositório paralelo `checkin-industrial-docs.wiki.git` a cada push em `main`.

Vantagem: edits via PR (com review) em vez de edição direta na UI do Wiki.

## Como configurar (one-time setup)

### 1. Secrets/vars no repo da API

- Variable `DOCKERHUB_USERNAME` = `checkinindustrial`
- Secret `DOCKERHUB_TOKEN` = access token gerado em hub.docker.com (Read + Write)

### 2. GitHub Pages para o Allure

Settings → Pages → Source: `Deploy from a branch` → Branch `gh-pages` `/` (root).

### 3. Wiki inicial

GitHub Wiki precisa de **uma página criada via UI** antes do workflow tomar conta:

1. No repo `checkin-industrial-docs` → aba **Wiki** → "Create the first page" → salvar qualquer conteúdo placeholder.
2. Próximo push na pasta `wiki/` do main dispara o workflow que sobrescreve com o conteúdo correto.

## Dependabot

Cada repo tem `.github/dependabot.yml` (criado nos PRs de production-readiness) atualizando:
- npm (painel)
- NuGet (api)
- pip (e2e)
- GitHub Actions (todos)

PRs do Dependabot têm CODEOWNERS apontando para os mantenedores; CI igual a qualquer PR humano.
