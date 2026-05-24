# Convenções de Código

Para profundidade, vá direto ao `CLAUDE.md` do repo específico. Esta página resume o que é **compartilhado** entre os 5 repos.

## CLAUDE.md por repo

Cada repositório tem um `CLAUDE.md` no root (ou em `src/` no caso da API) com:
- Stack + estrutura de pastas
- Convenções obrigatórias da feature/repo
- Anti-patterns que evitar
- Comandos comuns (build/test/lint/run)
- Troubleshooting

Links diretos:

- [API — `src/CLAUDE.md`](https://github.com/checkin-industrial/checkin-industrial-api/blob/main/src/CLAUDE.md)
  - [Features — `src/Features/CLAUDE.md`](https://github.com/checkin-industrial/checkin-industrial-api/blob/main/src/Features/CLAUDE.md)
  - [Empresas — `src/Features/Empresas/CLAUDE.md`](https://github.com/checkin-industrial/checkin-industrial-api/blob/main/src/Features/Empresas/CLAUDE.md)
  - [PontosInstitucionais — `src/Features/PontosInstitucionais/CLAUDE.md`](https://github.com/checkin-industrial/checkin-industrial-api/blob/main/src/Features/PontosInstitucionais/CLAUDE.md)
  - [TelefonesUteis — `src/Features/TelefonesUteis/CLAUDE.md`](https://github.com/checkin-industrial/checkin-industrial-api/blob/main/src/Features/TelefonesUteis/CLAUDE.md)
- [Painel — `CLAUDE.md`](https://github.com/checkin-industrial/checkin-industrial-painel/blob/main/CLAUDE.md)
- [E2E — `CLAUDE.md`](https://github.com/checkin-industrial/checkin-industrial-tests-e2e/blob/main/CLAUDE.md)

## Padrões cross-repo

### Soft delete

Implementado em todas as 3 features de CRUD. A entidade tem campo `Ativo: bool? default true`. O `DELETE` é idempotente — seta `Ativo=false` sem remover a linha.

Lado da API:
- `GET /<recurso>?ativo=true` filtra ativas; `?ativo=false` filtra inativas; sem parâmetro retorna todas.
- `PUT /<recurso>/{id}` aceita `ativo` no payload — admin pode reativar via update.

Lado do painel:
- Tela de Gestão tem dropdown "Somente ativas / Somente inativas / Todas".
- Coluna "Status" e botão condicional "Excluir" (se ativa) ou "Reativar" (se inativa).

### Convenção de queryKey no TanStack Query (painel)

```ts
queryKey: [FEATURE_KEY, ...variantes]
```

Exemplo: `["empresas", "list", statusFiltro]`. Invalidação após mutation:

```ts
await queryClient.invalidateQueries({ queryKey: ["empresas"] });
```

Zera todas as variantes da feature de uma vez.

### Naming de arquivos

| Tipo | Padrão | Exemplo |
|---|---|---|
| Endpoint na API | `<Verbo><Entidade>.cs` | `CreateEmpresa.cs`, `ListPontosInstitucionais.cs` |
| Sub-feature na API | Subpasta com `<SubFeature>Module.cs` | `Empresas/Importacao/ImportEmpresasModule.cs` |
| Componente React | PascalCase | `EmpresasManagementScreen.tsx` |
| Resource Robot | `<feature>_api.resource` | `empresas_api.resource` |

### Branch + PR

- **Branches**: `feat/<slug>`, `fix/<slug>`, `refactor/<slug>`, `test/<slug>`, `chore/<slug>`
- **Commits**: prefixo de tipo conforme branch (`feat:`, `fix:`, `refactor:`, ...). Mensagem em português.
- **PRs**: título resume o "porquê" em uma linha; descrição lista mudanças + test plan.
- **Co-Authored-By:** quando o PR foi gerado por IA, adicionar linha `Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>` no commit.

## Anti-patterns que evitamos

### Na API
- `[ApiController]` + controllers — migramos para Minimal APIs com `TypedResults<>`.
- Subpastas `Services/`, `Models/`, `Dtos/` dentro de uma feature — mantemos flat (VSA).
- Compartilhar DTOs entre features — duplicamos (verbosidade > acoplamento).
- `using AppTurismoIndustrial.Api.Features.*;` no topo de arquivos — global usings já cobrem.

### No painel
- Hard-coded URLs da API — usar `apiFetch()` que injeta `X-Api-Key` automaticamente.
- `useState` + `fetch` + `useEffect` para dados remotos — usar TanStack Query.
- `localStorage` para auth — usamos `sessionStorage` deliberadamente (banking-pattern).

### No E2E
- Hardcoded IDs entre testes — sempre criar a entidade no setup do teste.
- `Sleep 5s` para esperar consistência — usar `Wait Until Keyword Succeeds` com polling.
- Compartilhar suite variables entre tests — cada teste deve ser independente.
- Tocar diretamente no banco ou ler logs internos — validar só via API pública.

## Validação obrigatória antes de PR

| Repo | Comandos |
|---|---|
| API | `dotnet build` + `dotnet test` |
| Painel | `npm run typecheck` + `npm run lint` + `npm test` + `npm run build` |
| E2E | `python -m robot --dryrun tests/suites/` + suíte completa contra Docker Compose |
