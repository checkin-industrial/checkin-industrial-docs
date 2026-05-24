# Plano: Decompose do `EmpresasFilterMapExample` — Fase 2

Plano técnico no formato **Plan Mode** do Claude Code. Pode ser dado direto pra outro desenvolvedor (humano ou agente) executar.

> **Status**: Fase 1 completa. Fase 2 aguardando gatilho. Ver [Melhorias-Planejadas](Melhorias-Planejadas).

---

## Goal

Quebrar o componente monolítico [`EmpresasFilterMapExample.tsx`](https://github.com/checkin-industrial/checkin-industrial-painel/blob/main/src/features/empresas/EmpresasFilterMapExample.tsx) (~1526 linhas) em **5 sub-componentes coesos** + um container slim. Objetivo: deixar manutenção (por humano ou IA) viável sem ter que carregar todo o estado mental do mapa de uma vez.

Estado final esperado: container com ~300-400 linhas só com state global + composição; cada sub-componente com responsabilidade única.

## Contexto / estado atual

**Fase 1 (concluída)** — extraiu helpers em [`MapHelpers.tsx`](https://github.com/checkin-industrial/checkin-industrial-painel/blob/main/src/features/empresas/MapHelpers.tsx) (181 linhas: `MapViewport`, `MapFocusTarget`, `HeatmapLayer`, constantes geográficas, tipos auxiliares) e migrou 4 chamadas de fetch pra `useQuery` (PR [painel#14](https://github.com/checkin-industrial/checkin-industrial-painel/pull/14)).

**Fase 2 (este plano)** — extração de sub-componentes UI. Estado atual:

- Componente tem ~25 `useState` no topo + ~10 `useEffect`
- Painel direito de filtros (~350 linhas no JSX) é candidato a `FilterPanel`
- Popup de empresa (Tooltip do react-leaflet) é simples — talvez não compense extrair
- Bloco de vizinhança no painel de relatório direito (~200 linhas) é o `NeighborhoodOverlay`
- Polyline OSRM + indicador de duração é o `RouteOverlay`
- Footer com `{filteredCount} empresas filtradas` é o `StatsPanel`

## Decisões a tomar

### D1. Forma de compartilhar estado entre sub-componentes

- 🤔 **Opção 1**: prop drilling — passar callbacks + valores via props. Verbose mas explícito.
- 🤔 **Opção 2 (recomendação)**: **Context local** dentro do escopo do componente (`MapContextProvider` no topo, sub-componentes usam `useMapContext()`). Reduz boilerplate de prop drilling sem virar global state.
- alt: Zustand / atom-based (overkill pra escopo isolado)

**Critério de decisão**: se mais de 3 sub-componentes precisarem do MESMO state, Context vence. Pelo levantamento atual (filtros, vizinhança, rota compartilham `selectedEmpresaId`, `filters`, `layerToggles`), Context é o caminho certo.

### D2. Localização dos sub-componentes

- ✅ **`src/features/empresas/components/`** — subpasta nova. Mantém a feature autocontida e visualmente separa "tela principal" de "subpartes da tela".
- alt: flat em `features/empresas/` (poluição)
- alt: pasta separada por sub-componente (overkill pra peças pequenas)

### D3. Tipos TypeScript compartilhados

- ✅ **`features/empresas/types.ts`** — extrair tipos compartilhados (`EmpresaFilterMapItem`, `FilterFormState`, `LayerToggleState`, `ReportSectionKey`, etc.) do componente principal.
- Sub-componentes importam de lá.

### D4. Testes para sub-componentes

- ✅ **Adicionar 1 smoke test por sub-componente** seguindo o padrão de [`TelefonesUteisCardsScreen.test.tsx`](https://github.com/checkin-industrial/checkin-industrial-painel/blob/main/src/features/telefonesUteis/TelefonesUteisCardsScreen.test.tsx). Render simples + interaction básica.
- Cobertura aumenta naturalmente — antes era 0% pro mapa, viraria 5 testes pequenos.

### D5. Manter Context API simples vs construir hook customizado

- ✅ **Context + hook customizado** (`useMapContext()`) que faz o `useContext` + assert nullable. Pattern padrão React.

## Implementação passo-a-passo

### Fase 2a: Extrair tipos (mecânico, ~30min)

1. Criar `features/empresas/types.ts`
2. Mover types: `EmpresaFilterMapItem`, `EmpresaVizinhancaResponse`, `EmpresaVizinha`, `EmpresaVizinhancaBase`, `HeatmapPointApi`, `MapTargetPoint`, `FilterFormState`, `PontoInstitucionalFilterState`, `CnaeOption`, `LayerToggleState`, `ReportSectionKey`
3. Importar de `types.ts` no componente principal (deixar funcionar antes de continuar)
4. **Validação**: `npm run typecheck` + `npm run build` verdes

### Fase 2b: Criar MapContext (~1h)

1. `features/empresas/MapContext.tsx`:
   - `MapContextValue` agrupa todo o estado que será compartilhado:
     - `filters`, `setFilters`
     - `selectedEmpresaId`, `setSelectedEmpresaId`
     - `selectedPontoInstitucionalId`, `setSelectedPontoInstitucionalId`
     - `layerToggles`, `setLayerToggles`
     - `empresaBuscaAtiva`, `setEmpresaBuscaAtiva`
     - `vizinhanca` (do useQuery), `reportLoading`, `reportError`
     - `routePath`, `routeInfo`, `routeLoading`, `routeError`, `routeEnabled`
     - Refs: `mapStageRef`, `lastMapTargetRequestRef`
   - `useMapContext()` hook que faz assert
2. Refatorar `EmpresasFilterMapExample.tsx` pra envolver tudo num `<MapContextProvider value={...}>` mas SEM extrair sub-componentes ainda. Isso isola o risco da mudança de Context da mudança de estrutura.
3. **Validação**: testes E2E + build verdes (smoke browser opcional)

### Fase 2c: Extrair `FilterPanel` (~2h)

1. `features/empresas/components/FilterPanel.tsx`:
   - Aceita props **mínimas**: nada (consome do Context)
   - JSX = todo o `<aside className="map-right-sidebar">...</aside>` do componente atual (linhas ~862-1100, ~240 linhas)
   - Inclui: header com botão de minimizar, search input, sub-section Empresas (setor/porte/cnae/municipio/situacao), sub-section Pontos (termo/tipo), toggle de busca ativa/pontos ativos
2. Componente principal passa a renderizar `<FilterPanel />` no lugar do JSX inline
3. **Validação**: smoke browser — abrir/fechar painel, mudar filtros, confirmar que mapa atualiza
4. **Smoke test**: `FilterPanel.test.tsx` — render com Context mockado, click em "Aplicar", verifica callback chamado

### Fase 2d: Extrair `NeighborhoodOverlay` (~2h)

1. `features/empresas/components/NeighborhoodOverlay.tsx`:
   - Renderiza:
     - `<Circle>` no mapa indicando raio
     - `<Polyline>`s ligando base aos vizinhos
     - Painel de relatório direito (`<aside className="map-report-sidebar">`) com tabela de empresas próximas, mesmo CNAE, mesmo setor
   - Lê do Context: `selectedEmpresaId`, `vizinhanca`, `reportLoading`, `reportError`, `collapsedReportSections`
2. **Validação**: clicar numa empresa → painel aparece, círculo desenha, polylines aparecem

### Fase 2e: Extrair `RouteOverlay` (~1h)

1. `features/empresas/components/RouteOverlay.tsx`:
   - Renderiza:
     - `<Polyline>` da rota OSRM
     - Badge flutuante com "X km, Y min"
     - Botões "Limpar rota" / "Recalcular"
   - Lê do Context: `userLocation`, `rotaDestino`, `routePath`, `routeInfo`, `routeLoading`, `routeError`, `routeEnabled`
2. **Validação**: ativar localização → clicar empresa → calcular rota → verifica polyline

### Fase 2f: Extrair `StatsPanel` (~30min)

1. `features/empresas/components/StatsPanel.tsx`:
   - Renderiza footer com `{filteredCount} empresas filtradas | {pontosCount} pontos | etc.`
   - Lê do Context: `empresas`, `pontosInstitucionais`, `vizinhanca`

### Fase 2g: Extrair `EmpresaPopup` (opcional, ~1h)

Este é o mais discutível — o Tooltip do react-leaflet é simples (apenas markup, sem state próprio). Considerar:
- **Se** for trivial (5-10 linhas inline), manter inline.
- **Se** crescer com mais info (foto, link "ver detalhes", etc.), extrair.

Decisão fica pro momento da implementação. Não bloqueante.

### Fase 2h: Cleanup do container principal

1. Após extrações, `EmpresasFilterMapExample.tsx` deve ter ~300-400 linhas:
   - State management (useState, useQuery)
   - useEffects de sincronização
   - Composição: `<MapContextProvider>` + `<MapContainer>` + `<FilterPanel />` + `<NeighborhoodOverlay />` + `<RouteOverlay />` + `<StatsPanel />`
2. **Validação final**:
   - `npm run typecheck` + `npm run lint` + `npm test` + `npm run build` todos verdes
   - Smoke browser: percorrer fluxos golden — load inicial, filtro, click empresa, vizinhança, rota, heatmap toggle
   - E2E suite roda contra a stack: verde

## Estratégia de testes

| Camada | Cobertura | Ferramenta |
|---|---|---|
| Unit/component | 5 smoke tests (FilterPanel, NeighborhoodOverlay, RouteOverlay, StatsPanel, EmpresaPopup se extrair) | Vitest + Testing Library + Context mockado |
| Integração | Container renderiza tudo | Vitest com `MapContextProvider` real |
| E2E | Já existe — testes da API cobrem o backend, suite Robot não toca UI ainda | Robot (futuro: UI E2E com Browser library) |

## Riscos + edge cases

1. **Re-render performance**: Context updates re-renderiza todos os consumidores. Se virar gargalo, dividir em múltiplos Contexts (filters, selection, route) ou usar `useMemo` agressivo no `value`.
2. **react-leaflet hooks** (`useMap`, `useMapEvents`) só funcionam dentro de `<MapContainer>`. Sub-componentes que precisam disso devem ser filhos do `<MapContainer>`, não do `<MapContextProvider>`. O Provider precisa envolver AMBOS.
3. **Estado de filtro em useEffect chain**: a interação entre `filters`, `effectiveFilters`, `empresaBuscaAtiva` ainda está acoplada. Considerar consolidar em um `useFilters()` hook custom durante a fase 2c.
4. **Test isolation**: smoke tests do FilterPanel precisam de Context mockado com defaults sensatos. Criar um `mockMapContextValue()` helper em `features/empresas/components/__tests__/test-utils.tsx`.
5. **Bundle size**: pequeno aumento esperado (~3-5KB) por overhead de Context + chunking. Aceitável.

## Definition of Done

- [ ] `EmpresasFilterMapExample.tsx` ≤ 400 linhas
- [ ] `features/empresas/components/` tem ao menos 4 sub-componentes (FilterPanel + NeighborhoodOverlay + RouteOverlay + StatsPanel; EmpresaPopup opcional)
- [ ] 4-5 smoke tests novos no painel, todos verdes
- [ ] `npm run typecheck` + `lint` + `test` + `build` verdes
- [ ] Smoke browser: 6 fluxos golden manuais OK (descritos em [Para-Devs-Como-Rodar-Localmente](Para-Devs-Como-Rodar-Localmente))
- [ ] Suite E2E (`checkin-industrial-tests-e2e`) verde sem mudanças (a UI E2E não cobre este componente ainda — só a API)
- [ ] Item removido de [Melhorias-Planejadas](Melhorias-Planejadas)
- [ ] PR único OU PRs encadeados por fase (recomendado: 1 PR por fase 2a → 2h pra revisão menor e rollback fácil)

## Pano de fundo da decisão de adiamento

Adiado porque **não há custo concreto** de manter o componente como está enquanto não houver feature nova mexendo no mapa. O refactor pelo refactor adiciona risco (regressões em fluxos complexos do mapa) sem ROI imediato.

**Gatilho de retomada**: próxima feature que precise tocar 2+ áreas do componente (ex: filtro de tipo de empresa + popup com mais info + novo overlay). Aí o refactor + feature na mesma PR justifica.

Padrão técnico estabelecido em Fase 1 (PR [painel#14](https://github.com/checkin-industrial/checkin-industrial-painel/pull/14)) — extração de helpers via `MapHelpers.tsx` foi o cardinal point que provou a viabilidade. Fase 2 amplia o mesmo princípio.
