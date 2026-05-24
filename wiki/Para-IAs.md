# Para IAs (Claude Code)

Esta página é a porta de entrada explícita para **agentes de IA** que vão manter o sistema. Lê primeiro, depois pula pro repo certo.

## Princípio do projeto

> Este sistema passará a ter manutenção quase total por agentes de IA, portanto sua arquitetura deve refletir a melhor forma de trabalho para que um modelo de IA consiga entender os contextos rapidamente. A principal ferramenta para manutenção por IA será o Claude Code.

— *Premissa original do owner do projeto.*

Tudo no ecossistema reflete isso: VSA, soft delete consistente, CLAUDE.md em cada feature, convenções explícitas, sem "magic".

## Ordem de leitura recomendada

Para tarefas grandes (nova feature, refactor cross-repo):

1. Esta Wiki — começa por **[Arquitetura](Para-Devs-Arquitetura)** + **[Convenções](Para-Devs-Convencoes)**
2. `CLAUDE.md` do repo onde a tarefa mora
3. `CLAUDE.md` da feature específica (em `src/Features/<X>/CLAUDE.md` na API)

Para tarefas pequenas (fix bug, small change):

1. `CLAUDE.md` da feature direto
2. Esta Wiki só se a tarefa cruzar limites de repo

## Onde estão os CLAUDE.md

### API

- [`src/CLAUDE.md`](https://github.com/checkin-industrial/checkin-industrial-api/blob/main/src/CLAUDE.md) — root tech stack + estrutura + scripts
- [`src/Features/CLAUDE.md`](https://github.com/checkin-industrial/checkin-industrial-api/blob/main/src/Features/CLAUDE.md) — convenção de VSA + como adicionar feature/endpoint novo
- [`src/Features/Empresas/CLAUDE.md`](https://github.com/checkin-industrial/checkin-industrial-api/blob/main/src/Features/Empresas/CLAUDE.md)
- [`src/Features/PontosInstitucionais/CLAUDE.md`](https://github.com/checkin-industrial/checkin-industrial-api/blob/main/src/Features/PontosInstitucionais/CLAUDE.md)
- [`src/Features/TelefonesUteis/CLAUDE.md`](https://github.com/checkin-industrial/checkin-industrial-api/blob/main/src/Features/TelefonesUteis/CLAUDE.md)

### Painel

- [`CLAUDE.md`](https://github.com/checkin-industrial/checkin-industrial-painel/blob/main/CLAUDE.md) — root stack + auth + TanStack Query pattern
- [`src/features/empresas/CLAUDE.md`](https://github.com/checkin-industrial/checkin-industrial-painel/blob/main/src/features/empresas/CLAUDE.md)
- [`src/features/pontosInstitucionais/CLAUDE.md`](https://github.com/checkin-industrial/checkin-industrial-painel/blob/main/src/features/pontosInstitucionais/CLAUDE.md)
- [`src/features/telefonesUteis/CLAUDE.md`](https://github.com/checkin-industrial/checkin-industrial-painel/blob/main/src/features/telefonesUteis/CLAUDE.md)

### E2E

- [`CLAUDE.md`](https://github.com/checkin-industrial/checkin-industrial-tests-e2e/blob/main/CLAUDE.md) — comandos, convenções Robot, padrão BDD opcional

## Workflow recomendado para agentes

### 1. Antes de modificar código

- Identifique o repo certo. Cruze com a [Arquitetura](Para-Devs-Arquitetura) se não tiver certeza.
- Leia o `CLAUDE.md` da feature. Os anti-patterns ali não são opcionais.
- Para mudanças cross-repo (API + painel + E2E), planeje **PRs encadeados** — não tente fazer tudo em um.

### 2. Durante a mudança

- Use `TypedResults<>` em endpoints novos da API (não `IActionResult`).
- Use `useQuery` + `apiFetch` em chamadas novas do painel (não `useEffect` + `fetch`).
- Adicione um teste E2E se a mudança expõe um endpoint novo ou muda contrato.

### 3. Antes do PR

| Repo | Comandos |
|---|---|
| api | `dotnet build` + `dotnet test` |
| painel | `npm run typecheck` + `npm run lint` + `npm test` + `npm run build` |
| e2e | `python -m robot --dryrun` + suíte completa local contra Docker Compose |

### 4. No PR

- Título: prefixo de tipo + escopo + 1 linha em português.
- Descrição: lista mudanças por categoria, test plan checklist.
- Co-author tag: `Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>` no commit.
- Mergem encadeado: API primeiro → aguarda publish Docker Hub → painel + E2E em paralelo.

## Princípios não-óbvios

### Soft delete e consistência

Toda entidade soft-deleta. Hard delete foi removido (era inconsistência entre features). Antes de adicionar uma feature CRUD nova, espelhe o padrão de Telefones/Empresas/Pontos.

### Output cache e mutations

A API faz output cache de reads (TTL 60s default). Após uma mutation, o **painel** invalida via TanStack Query (`invalidateQueries`). O **E2E** desliga o cache via env (`OutputCache__ReadEndpointTtlSeconds=0`) para evitar flakiness em assertions imediatas.

### Auth fail-fast em produção

Em `Production`, o startup da API aborta se `Auth__ApiKey` ou `Cors__AllowedOrigins` não estiverem configurados. Em `Development`, libera tudo com warning. Não tente "consertar" essa diferença — é deliberada.

### Robot enums e tipos

Robot's `Create Dictionary` armazena valores como string por default. JSON enums da API (e.g., `Setor`, `Categoria`, `Tipo`) precisam de `${{int($var)}}` para serializar como int. Lat/long usam `${{float($var)}}`. Vide [`empresas_api.resource`](https://github.com/checkin-industrial/checkin-industrial-tests-e2e/blob/main/tests/resources/keywords/empresas_api.resource).

## Quando NÃO confiar nesta Wiki

A Wiki é alto nível e pode ficar desatualizada relative ao código. Se for citar um arquivo, uma linha específica, uma versão de stack, ou um comportamento de runtime, **verifique no código primeiro**. O `git log` é a verdade.

## Memórias de sessões anteriores

Agentes Claude Code mantêm memórias locais sob `~/.claude/projects/c--git-checkin-industrial/memory/`. Indexadas em `MEMORY.md`. Verificar antes de propor mudanças grandes — pode haver decisão prévia já tomada (e.g., feature adiada por billing).
