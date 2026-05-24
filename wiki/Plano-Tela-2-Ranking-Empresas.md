# Plano: Tela 2 — Ranking de Empresas (Top + CNAEs)

> **Leia primeiro**: [Plano-Relatorios-Overview](Plano-Relatorios-Overview) — define fundações compartilhadas (F1-F4) e perguntas abertas (Q1-Q4) que esta tela depende.

---

## Goal

Tela analítica que mostra **Top 10 empresas por faturamento** (com mapa numerado + tabela + export) e **Top CNAEs mais frequentes** (barra horizontal proporcional + tabela). É o relatório "executivo" que sintetiza o cadastro em métricas acionáveis.

## Contexto

- A spec original previa essa tela como "gerencial principal" — Top 10 + CNAEs + ranking exportável.
- Stakeholder de destino: prefeitura, secretaria de desenvolvimento econômico, consultor analista — quer "ver as 10 maiores" e "quais setores predominam" sem mexer em filtros complexos.
- Diferencia-se da Tela 3 por **não ter filtros pesados** — é leitura, não exploração. Filtros mínimos (período, talvez setor).

## Pré-requisitos (cross-plan)

- ⚠️ **Q1 respondida** — fonte de faturamento. Sem isso, "ranking por faturamento" não tem dado. Mitigação temporária: ranking por `NumeroFuncionarios` (proxy) enquanto Q1 não fecha, com label honesto "Top por # funcionários (faturamento em breve)".
- ✅ **F1** (Recharts) — para a barra horizontal de CNAEs.
- ✅ **F3** (Filter DTO std) — recomendado mas não bloqueante.

## Estado atual

- Backend tem endpoint de filtro de empresas, mas **sem agregações** (`GROUP BY` em CNAE, ranking ordenado).
- Frontend não tem componente de gráficos. Recharts entra como nova dependência.
- Export CSV: padrão simples, não temos ainda na base.

## Decisões a tomar

### D1. Métrica primária do ranking ⭐ recomendação marcada

Opções:
- (a) Faturamento (depende de Q1).
- (b) # Funcionários (proxy, dados temos).
- (c) Configurável por usuário (combo "ordenar por").

**Recomendação: (c) configurável**, com defaults sequenciais: lança com "# Funcionários" como default; quando Q1 entregar, troca default pra "Faturamento". Implementação igual de qualquer forma — pôr a `ordenarPor` no contrato desde o início evita migration.

### D2. Top N: fixo em 10 ou paramétrico?

Opções:
- (a) Fixo em 10 — simplifica UI.
- (b) Combo "Top 10/25/50/100".
- (c) Paginação ilimitada.

**Recomendação: (b).** Match com spec original (10) mas dá fôlego pra ranking maior sem perder simplicidade. (c) seria over-engineering pra uma tela de "top".

### D3. Período/timeframe

Opções:
- (a) Sem filtro de período — ranking instantâneo do cadastro atual.
- (b) "Últimos N meses" baseado em `CriadoEm` da empresa — mostra novas entradas.
- (c) Período aplicado ao cálculo de faturamento (anual, mensal).

**Recomendação: (a) inicial.** Faturamento não tem dimensão temporal modelada ainda. Quando tiver, (c) pode entrar. (b) é métrica de crescimento, não de ranking — entra em outra tela.

### D4. Export: formato e gatilho

Opções:
- (a) Botão "Exportar CSV" — download imediato.
- (b) Botão "Gerar Relatório" que cria PDF formatado (com header, logo, summary).
- (c) Ambos.

**Recomendação: (c).** CSV é trivial, serve workflows analíticos. PDF é o "produto final" pra apresentação. Implementar CSV primeiro, PDF em PR separado.

### D5. CNAE: classe ou subclasse?

CNAE tem hierarquia (Seção → Divisão → Grupo → Classe → Subclasse). Ex: "47.71-7/02 - Comércio varejista de artigos do vestuário e acessórios".

Opções:
- (a) Mostrar subclasse completa (7 dígitos) — granular mas confuso.
- (b) Agregar por Classe (5 dígitos) — equilíbrio.
- (c) Agregar por Divisão (2 dígitos) — bird's eye.
- (d) Combo configurável.

**Recomendação: (d) com default em Classe (b).** Aderente ao que IBGE expõe e ao que faz sentido pra leigos.

## Implementação passo-a-passo

### Backend — `Features/Relatorios/`

Nova feature `src/Features/Relatorios/` na API.

#### 1. Endpoint ranking de empresas

```
GET /api/relatorios/ranking-empresas
  ?ordenarPor=funcionarios|faturamento
  &limite=10|25|50|100
  &setor=METALURGIA|...   (opcional)
- Auth: anônimo + cache 5min
- Returns:
    [{
      posicao: 1,
      empresaId: guid,
      razaoSocial: "...",
      latitude, longitude,
      valor: 145,           // # funcionários OU faturamento
      cnaeDescricao: "..."
    }]
```

Implementação: query EF `OrderByDescending(ordenarPor) Take(limite)` com filtro de `Ativo=true`.

#### 2. Endpoint CNAEs mais frequentes

```
GET /api/relatorios/cnaes-mais-frequentes
  ?agruparPor=subclasse|classe|divisao
  &limite=10
- Auth: anônimo + cache 5min
- Returns:
    [{
      codigo: "47.71-7",
      descricao: "Comércio varejista de artigos do vestuário",
      total: 142,
      percentual: 18.3       // % do total de empresas
    }]
```

Implementação: `GROUP BY` no SQL via `IGrouping`, ordenado por count desc.

#### 3. Endpoint export CSV

```
GET /api/relatorios/ranking-empresas/csv?<mesmos params>
- Auth: API Key (export é write-y, auditável)
- Returns: text/csv com header padrão
```

Implementação: gera CSV em memória (poucos KB, < 100 linhas). Usar `CsvHelper` se já não estiver no projeto, ou serializar à mão (simples).

#### 4. Endpoint export PDF (D4 — fase 2 do PR)

```
POST /api/relatorios/gerar-pdf
  Body: { tipo: "ranking-empresas", filtros: {...} }
- Auth: API Key
- Returns: application/pdf
```

Implementação: usar `QuestPDF` (community license se < 1M USD revenue, free pra esse caso). PR separado — não bloqueia entrega CSV.

### Frontend — `features/relatorios/`

Estrutura proposta:

```
src/features/relatorios/
├── CLAUDE.md
├── RankingEmpresasScreen.tsx           (tela Tela 2)
├── api/
│   ├── useRankingEmpresas.ts           (useQuery)
│   ├── useCnaesMaisFrequentes.ts       (useQuery)
│   ├── useExportarRanking.ts           (download CSV)
│   └── useGerarPdfRanking.ts           (geração + download)
├── components/
│   ├── RankingTable.tsx                (tabela genérica c/ export — F3 dos shared do Overview)
│   ├── CnaesBarChart.tsx               (Recharts BarChart horizontal)
│   ├── MapaTop10.tsx                   (Leaflet com markers numerados)
│   └── EmpresaMarkerNumerado.tsx       (ícone customizado com badge da posição)
└── RankingEmpresasScreen.test.tsx
```

### Layout

```
┌────────────────────────────────────────────────────────────────┐
│ Ranking de Empresas                                            │
│                                                                │
│ [Ordenar por: Faturamento ▾] [Top 10 ▾] [Setor: Todos ▾]       │
│ [📥 Exportar CSV] [📄 Gerar Relatório]                          │
├──────────────────────────────┬─────────────────────────────────┤
│ Mapa Top 10 (markers 1-10)   │ Top CNAEs (barras horizontais)  │
│                              │                                 │
│   [react-leaflet]            │   [Recharts BarChart]           │
│                              │                                 │
├──────────────────────────────┴─────────────────────────────────┤
│ Tabela: # | Razão Social | CNAE | Funcionários | Faturamento   │
│  1 | Empresa A | 47.71-7 | 145 | R$ 12M                        │
│  2 | ...                                                        │
└────────────────────────────────────────────────────────────────┘
```

### Decisões UI

- Markers numerados: posição visível dentro do círculo (1-10 cabe; >10 vira "11+").
- Hover/click em marker do mapa: highlight da linha correspondente na tabela.
- Hover na barra de CNAE: tooltip com count exato + percentual.
- Cores: paleta sequencial pra ranking (verde → amarelo → vermelho indicando posição).

## Estratégia de testes

| Camada | Cobertura |
|---|---|
| Unit (API) | `RankingService.GetTopN(ordenarPor, limite)`, `CnaeAggregator.GroupBy(nivel)` |
| Integration (EF) | Query com diferentes `ordenarPor` retorna sorted correto + respeita `Ativo=true` |
| E2E (Robot) | Nova suite `05__relatorios.robot`: rank top 10, GROUP BY CNAE, export CSV (download + content type), filtros combinados |
| UI (Vitest) | Smoke: tela renderiza com mock; ordenação muda dispara refetch; export dispara download |
| Manual | Verificar # funcionários consistente entre tabela e tooltip do marker |

## Riscos + edge cases

1. **Empresas sem CNAE preenchido** — agrupador GROUP BY: agregar em "Não classificado" ou excluir? Recomendação: incluir como categoria explícita.
2. **CNAEs órfãos** (código sem descrição na tabela CNAEs) — fallback "Código CNAE não encontrado (47.71-7)".
3. **Empate em ranking** — ordenação secundária por `RazaoSocial` ASC (determinístico).
4. **Cache stale após CRUD** — invalidar cache quando empresa é criada/atualizada/inativada. Mitigação: cache curto (5min) suficiente.
5. **CSV com vírgulas em razão social** — quote escaping correto (RFC 4180). Lib `CsvHelper` resolve.
6. **Export pesado** — limite 100. Mais que isso vira "exportar lista completa" em outro endpoint paginado.

## Definition of Done

- [ ] Q1 respondida (faturamento) OU decisão de lançar com `# funcionários` como métrica primária
- [ ] F1 (Recharts) instalado e funcionando
- [ ] 3 endpoints novos (`/ranking-empresas`, `/cnaes-mais-frequentes`, `/csv`) com testes E2E verdes
- [ ] Tela `RankingEmpresasScreen` com 3 painéis (mapa + barras + tabela) sincronizados
- [ ] Export CSV funciona via browser download
- [ ] Smoke browser: trocar ordenação muda mapa + tabela + barras consistentemente
- [ ] `CLAUDE.md` da feature `Relatorios`
- [ ] PDF deixado em PR de follow-up se Q1 ainda aberta
- [ ] Item movido de [Melhorias-Planejadas](Melhorias-Planejadas) pra histórico

## Estimativa

| Fase | Esforço |
|---|---|
| Backend (`RankingService` + `CnaeAggregator`) | 0.5 dia |
| Backend (export CSV) | 0.25 dia |
| Frontend (3 componentes + tela) | 1.5 dias |
| Testes E2E + Vitest | 0.5 dia |
| **Total (CSV only)** | **~2.75 dias** |
| PDF (PR follow-up) | +0.5 dia |
