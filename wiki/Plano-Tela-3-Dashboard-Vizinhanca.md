# Plano: Tela 3 — Dashboard Geoespacial / Análise de Vizinhança

> **Leia primeiro**: [Plano-Relatorios-Overview](Plano-Relatorios-Overview) — define fundações compartilhadas (F1-F4) e perguntas abertas (Q1-Q4) que esta tela depende.

---

## Goal

Tela de **exploração interativa** que combina filtros densos (porte, setor, situação, faixa de faturamento), camadas geográficas opcionais (distritos, heatmap, rodovias), e — ao selecionar uma empresa — abre painel de **análise de vizinhança** (KPIs de raio configurável, lista de vizinhos, opcionalmente fornecedores/concorrentes).

É a evolução natural do `EmpresasFilterMapExample` público — adiciona dimensões analíticas pra o usuário avançado (analista, consultor).

## Contexto

- Há sobreposição clara com o `EmpresasFilterMapExample` já em produção. **Não duplicar a tela**: estender ou criar variante "modo analítico".
- O painel já implementa: filtros básicos, mapa, heatmap, sub-mapa de vizinhança ao clicar em empresa, rota OSRM. Falta: filtros avançados (faixa faturamento, situação CNPJ ativa/inativa, múltipla seleção), camadas de distrito, KPIs estruturados de vizinhança.
- A "Fase 2" do decompose ([Plano-Decompose-EmpresasFilterMap-Fase-2](Plano-Decompose-EmpresasFilterMap-Fase-2)) **deveria acontecer antes** dessa tela — mexer num componente de 1500 linhas pra adicionar novas features é frágil.

## Pré-requisitos (cross-plan)

- ⚠️ **[Plano-Decompose-EmpresasFilterMap-Fase-2](Plano-Decompose-EmpresasFilterMap-Fase-2) — fortemente recomendado antes**. Adicionar features ao componente atual sem decompose vai degradar manutenibilidade. Não bloqueante, mas combinar é o uso ótimo do refactor.
- ⚠️ **Q1 respondida** — sem faturamento, slider de faixa não tem dado. Mitigação: lançar sem filtro de faturamento se Q1 ainda aberta.
- ⚠️ **Q2 respondida** — "Fornecedor/Concorrente" depende. **Recomendação: adiar essa seção** (Opção 4 de Q2), entregar Tela 3 sem ela.
- ✅ **F1** (Recharts) — pra mini-gráficos do painel de vizinhança (distribuição de porte dos vizinhos).
- ✅ **F2** (Cluster markers) — densidade vai exigir.
- ✅ **F3** (Filter DTO std) — Tela 3 dobra # de filtros, padronização paga em manutenibilidade.

## Estado atual

- `EmpresasFilterMapExample.tsx` (~1500 linhas) tem filtros simples + heatmap + popup + sub-mapa.
- Endpoint `/api/empresas/{id}/neighbors?raioMetros=N` já existe (entregue na PR de soft delete).
- Faltam: agregação de KPIs (não só lista, mas counts por porte/setor), camada de distritos.

## Decisões a tomar

### D1. Nova tela ou modo "avançado" do mapa atual? ⭐ recomendação marcada

Opções:
- (a) Nova rota `/dashboard` com componente novo.
- (b) Mesmo componente, toggle "Modo Análise" que mostra/esconde painéis avançados.
- (c) Mesma rota, layout responsivo com tudo sempre visível (painéis dobráveis).

**Recomendação: (a) nova rota** — separação clara de responsabilidades. O mapa público continua o que é (descoberta básica). Dashboard é "modo analista". Reutiliza sub-componentes via shared, mas é tela própria.

### D2. Filtros: form vs sliders inline

Spec sugere "filtros laterais": porte (radio), setor (multi-select), situação (radio Ativa/Inativa), faturamento (slider 0M-100M).

Opções:
- (a) Painel lateral fixo com todos os controles.
- (b) Bottom drawer (mobile-friendly) que sobe ao tap em "Filtros".
- (c) Floating toolbar com filtros mais usados + "Avançado" expansível.

**Recomendação: (a) painel lateral fixo no desktop, (b) bottom drawer no mobile** — máxima visibilidade em desktop (usuário é analista, tela grande), mobile usa pattern conhecido.

### D3. Faixa de faturamento: como representar?

Opções:
- (a) Dois inputs numéricos "De R$ ___ Até R$ ___".
- (b) Range slider (dual handle).
- (c) Combo com faixas pré-definidas ("Até 360k MEI", "360k-4.8M ME", etc.).

**Recomendação: (b) range slider** + tooltips mostrando valor — UX explorativa precisa de feedback contínuo. (c) é mais "negócio" mas restringe demais.

### D4. KPIs de vizinhança: o que mostrar?

Quando usuário seleciona empresa focada, painel de vizinhança mostra:

Opções a incluir:
- Count total de vizinhos no raio
- Distribuição por porte (mini bar chart)
- Distribuição por setor (top 3 + "outros")
- Distância média / mediana
- Empresa mais próxima
- Densidade (empresas/km²)

**Recomendação:** primeiros 4 KPIs no painel inicial; "Empresa mais próxima" e "Densidade" como secundários expandíveis. Mais que isso vira ruído.

### D5. Persistência de filtros

Opções:
- (a) Stateful no componente — perde ao navegar fora.
- (b) URL query params — compartilhável + back/forward browser.
- (c) localStorage — persiste entre sessões.

**Recomendação: (b) URL params**. Analistas compartilham link "veja isso filtrado por X". Padrão moderno. Mais investimento mas justifica.

## Implementação passo-a-passo

### Pré-trabalho: Decompose Fase 2 (se ainda não feito)

Executar [Plano-Decompose-EmpresasFilterMap-Fase-2](Plano-Decompose-EmpresasFilterMap-Fase-2) primeiro. Resulta em:

```
features/empresas/mapa/
├── EmpresasFilterMapExample.tsx       (mantém — orquestrador)
├── components/
│   ├── FilterPanel.tsx
│   ├── EmpresaPopup.tsx
│   ├── NeighborhoodOverlay.tsx
│   ├── RouteOverlay.tsx
│   └── StatsPanel.tsx
└── ...
```

Esses componentes são **reaproveitados** pela Tela 3.

### Backend

#### 1. Endpoint análise de vizinhança extendido

```
GET /api/empresas/{id}/analise-vizinhanca?raioMetros=5000
- Auth: anônimo + cache 5min
- Returns:
    {
      empresaFocada: { id, razaoSocial, lat, long },
      raioMetros: 5000,
      kpis: {
        totalVizinhos: 42,
        distanciaMediaMetros: 1234,
        distanciaMedianaMetros: 1100,
        densidadeKm2: 0.53,
        empresaMaisProxima: { id, distanciaMetros },
        distribuicaoPorPorte: { "MEI": 12, "ME": 18, ... },
        distribuicaoPorSetor: { "Metalurgia": 15, ... }
      },
      vizinhos: [{
        empresaId, razaoSocial, lat, long,
        distanciaMetros, porte, setor
      }]
    }
```

Este é uma evolução do `/neighbors` existente. Backwards-compatibility: o endpoint atual continua válido, o novo é endpoint distinto.

#### 2. Endpoint de filtro avançado de empresas

```
GET /api/empresas/filter-avancado
  ?porte=MEI&porte=ME&porte=EPP
  &setor=METALURGIA
  &situacao=ativa
  &faturamentoMin=0&faturamentoMax=10000000
  &distritoId=guid
- Auth: anônimo + cache 1min
- Returns: [{ id, razaoSocial, lat, long, porte, setor, faturamento }]
```

Reaproveitar `EmpresaFilterParams` (após F3 — padronização). Suportar array em `porte` (multi-select).

### Frontend — `features/dashboard/`

Estrutura:

```
src/features/dashboard/
├── CLAUDE.md
├── DashboardScreen.tsx
├── api/
│   ├── useFilterAvancado.ts
│   └── useAnaliseVizinhanca.ts
├── components/
│   ├── FiltrosLaterais.tsx           (D2)
│   ├── CamadasControl.tsx            (toggle distritos, heatmap, rotas)
│   ├── PainelVizinhanca.tsx          (KPIs + mini charts)
│   ├── KpiCard.tsx                   (component genérico — extrair pra shared depois)
│   └── DistribuicaoPorteChart.tsx
└── DashboardScreen.test.tsx
```

### Layout

```
┌──────────────────────────────────────────────────────────────────────┐
│ Dashboard Geoespacial                          [URL: ?porte=ME&...]   │
├─────────────────┬────────────────────────────────────────────────────┤
│ Filtros         │                                                    │
│                 │                                                    │
│ Porte           │         Mapa com filtros aplicados                 │
│ □ MEI           │         + camadas ativas                           │
│ □ ME            │         + cluster markers (F2)                     │
│ ...             │                                                    │
│                 │                                                    │
│ Setor (multi)   │      [Selecionar empresa pra ver vizinhança]      │
│ ...             │                                                    │
│                 ├────────────────────────────────────────────────────┤
│ Faturamento     │ Painel Vizinhança (aparece ao selecionar empresa) │
│ [══●═══●═════]  │ ──────────────────────────────────────────────────  │
│ 0M  10M  100M   │ Empresa Focada: ABC LTDA                            │
│                 │                                                     │
│ Camadas         │ [42 Vizinhos] [1.2km Média] [0.53/km² Densidade]    │
│ □ Distritos     │                                                     │
│ □ Heatmap       │ Distribuição Porte: [██████ MEI / ████ ME / █ EPP]  │
│ □ Rodovias      │                                                     │
│                 │ Lista de Vizinhos: [tabela ordenável por distância] │
├─────────────────┴────────────────────────────────────────────────────┤
```

### Decisões UI

- Filtros: debounce 300ms antes de disparar query (evitar refetch em cada keystroke).
- Filtros multi-select com chips removíveis.
- Mapa: ao selecionar empresa, marker dela vira destacado + raio desenhado em transparência.
- Painel vizinhança: collapsible (default open quando empresa selecionada).
- URL params sync: filtros → `pushState` (não recarregar), mudança de URL → atualiza filtros.

## Estratégia de testes

| Camada | Cobertura |
|---|---|
| Unit (API) | `AnaliseVizinhancaService.Calcular(empresaId, raio)`, distribuições, KPIs determinísticos |
| Integration (EF) | Filtro avançado com array de portes funciona; range de faturamento |
| E2E (Robot) | Nova suite `06__analise-vizinhanca.robot`: KPIs corretos pra cenário sintético com 10 vizinhos; filtro multi-porte; soft-deleted empresa não aparece |
| UI (Vitest) | Smoke: tela renderiza, filtros aplicam, painel vizinhança aparece ao mockar select |
| Manual | URL share funciona (copiar URL com filtros → novo browser carrega no mesmo estado); performance com 500+ empresas no mapa (cluster F2 ativa) |

## Riscos + edge cases

1. **Empresa focada sem vizinhos no raio** — KPIs todos zero/null; UI mostra "Sem vizinhos no raio selecionado" em vez de quebrar charts.
2. **Filtros muito restritivos** → 0 resultados — UI mostra "Nenhuma empresa encontrada com esses filtros" + link "Limpar filtros".
3. **Performance com 1000+ empresas** — cluster F2 ativa automaticamente; sem cluster, latência aceitável até ~200.
4. **URL params muito longos** (multi-select de 20 setores) — limite de URL (~2000 chars) é folgado pra esse caso.
5. **Race condition de query** — filtros mudam rápido, queries antigas chegam tarde. TanStack Query já trata via abort de requests obsoletos.
6. **Compartilhar URL com `?ativo=false`** — usuário externo vê inativas. Considerar restringir esse param no filter público (só admin).
7. **Heatmap + cluster simultaneamente** — sobreposição visual. Recomendação: toggle exclusivo (escolhe um ou outro).

## Definition of Done

- [ ] Fase 2 do decompose entregue OU justificativa explícita de seguir sem
- [ ] F1, F2, F3 entregues
- [ ] Q1, Q2 respondidas (faturamento, fornecedor/concorrente)
- [ ] Endpoint `/api/empresas/{id}/analise-vizinhanca` com KPIs implementado e testado
- [ ] Endpoint `/api/empresas/filter-avancado` com multi-porte e range de faturamento
- [ ] `DashboardScreen` com filtros sincronizados a URL params
- [ ] Painel de vizinhança mostra 4 KPIs primários
- [ ] Distribuições renderizadas em Recharts
- [ ] Cluster markers ativa automaticamente com >200 empresas
- [ ] Smoke browser: filtrar por setor + porte → mapa atualiza → selecionar empresa → painel vizinhança aparece → URL share funciona
- [ ] `CLAUDE.md` da feature `Dashboard`
- [ ] Item movido de [Melhorias-Planejadas](Melhorias-Planejadas) pra histórico

## Estimativa

| Fase | Esforço |
|---|---|
| Pré-trabalho: Decompose Fase 2 | 2-3 dias (se ainda não feito — plano próprio) |
| Backend (`AnaliseVizinhancaService` + filter avançado) | 1.5 dia |
| Frontend (filtros laterais + camadas) | 1 dia |
| Frontend (painel vizinhança + KPIs) | 1 dia |
| URL params sync + edge cases | 0.5 dia |
| Testes E2E + Vitest | 1 dia |
| Manual smoke + ajustes UX | 0.5 dia |
| **Total (sem Decompose)** | **~5.5 dias** |
| **Total (com Decompose Fase 2 antes)** | **~8-9 dias** |

## Possível faseamento de entrega

Se o escopo de 8-9 dias for muito grande:

- **Fase A (~3.5d)**: Tela 3 base — nova rota + filtros laterais + mapa com filtros + sem painel vizinhança.
- **Fase B (~2d)**: Painel vizinhança com 4 KPIs.
- **Fase C (~1d)**: Camadas (distritos + rodovias).
- **Fase D (~1d)**: URL params sync + cluster F2.

Cada fase entrega valor isolável; B/C/D podem rodar em paralelo após A.
