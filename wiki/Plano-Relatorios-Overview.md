# Plano: Relatórios Gerenciais — Overview

Plano de **fundações compartilhadas** entre as 3 telas de relatórios gerenciais. Cada tela tem plano técnico próprio:

- [Plano-Tela-1-Configuracao-Mapas](Plano-Tela-1-Configuracao-Mapas)
- [Plano-Tela-2-Ranking-Empresas](Plano-Tela-2-Ranking-Empresas)
- [Plano-Tela-3-Dashboard-Vizinhanca](Plano-Tela-3-Dashboard-Vizinhanca)

---

## Goal

Entregar 3 telas de **inteligência territorial** (configuração GIS administrativa, ranking analítico, dashboard de vizinhança) que ampliam a área de gestão atual com capacidades de relatório e análise. Sem rewrite do produto — adaptação sobre o stack já consolidado (React 19 + .NET 10 + Postgres + react-leaflet).

## Reconciliação spec ↔ código atual

A especificação recebida sugere stack diferente do nosso. Posição padrão: **adaptar a spec ao stack existente**, flagando as divergências.

| Aspecto da spec | Spec sugere | Já temos | Decisão |
|---|---|---|---|
| Backend | NestJS | .NET 10 + EF Core | **Manter .NET 10** |
| DB spatial | PostgreSQL + PostGIS | Postgres puro + Haversine em C# | **Manter Haversine** no server + Turf.js no cliente UI-only quando precisar de geometria complexa (não migrar pra PostGIS agora) |
| CSS | TailwindCSS + Shadcn UI | CSS global em `styles.css` | **Manter CSS atual** — Tailwind/Shadcn é refactor próprio, separado dessas 3 telas (já está no backlog) |
| Charts | Recharts ou ECharts | Não temos lib de gráficos | **Adotar Recharts** — ainda não há outra escolha; é nova adição válida |
| Map cluster | clustering necessário | Sem clustering hoje | **Adotar `react-leaflet-cluster`** quando o N de empresas justificar (>200 markers visíveis) |
| Auth | JWT + RBAC | API Key + sessionStorage | **Manter API Key** — Telas 2/3 são públicas (anônimas com cache); Tela 1 é admin. Se RBAC for necessário pra granularidade fina (e.g., "user pode ler relatórios mas não importar"), aí planeja separado |
| State management | Zustand sugerido | TanStack Query + Context | **Manter TanStack Query** |

## Perguntas abertas que **bloqueiam** execução das telas

⚠️ **Estas perguntas precisam de resposta do owner antes de implementar.** Cada uma afeta múltiplas telas.

### Q1. Faturamento estimado (Telas 2 e 3)

Hoje `Empresa` tem `NumeroFuncionarios` mas **não tem `Faturamento`**. As Telas 2 e 3 mostram faturamento (ranking + filtro de range 0M-100M).

**Opções pra fonte do dado:**
1. **Manual** — admin preenche no cadastro/import CSV. Simples mas trabalhoso e desatualiza rápido.
2. **Faixa derivada do Porte** — MEI/ME/EPP/LTDA/SA têm faixas de receita legais. Heurística, baixa precisão mas zero esforço por empresa.
3. **Integração externa** — Receita Federal não expõe; haveria provedores comerciais (Serasa, Bigdata) — custo + complexidade.
4. **Estimativa por # funcionários × CNAE** — algoritmo simples (média setorial × headcount), heurística melhor que Porte.

Recomendação técnica: **Opção 4 calculada server-side**, com Opção 1 como override manual. Mas é decisão de produto, não técnica.

### Q2. Lógica de "Fornecedor" vs "Concorrente" (Tela 3)

A Tela 3 lista "Maiores Fornecedores" e "Principais Concorrentes" da empresa focada. Hoje **não temos a relação modelada** entre empresas.

**Opções:**
1. **Concorrente = mesmo CNAE; Fornecedor = CNAE complementar** — precisaríamos de uma tabela "CNAE → CNAEs complementares" (potencialmente externa, IBGE).
2. **Relação explícita cadastrada** — admin marca quem é fornecedor/cliente de quem. Trabalhoso mas preciso.
3. **Heurística por proximidade + setor** — Concorrente = mesmo setor próximo; Fornecedor = setor "anterior" (matéria-prima) próximo. Lossy.
4. **Adiar essa seção da Tela 3** — entregar apenas a parte de proximidade/KPIs primeiro; "Fornecedores/Concorrentes" entra em PR futuro com modelo de dados resolvido.

Recomendação: **Opção 4** — entregar a Tela 3 sem essa seção primeiro. Modelar a relação fornece dados sólidos depois.

### Q3. Settings dinâmicos: nova entidade (Tela 1)

A Tela 1 permite ao admin trocar tile server, upload de GeoJSON de distritos, controle de opacidade do heatmap. Hoje essas configs vivem em **env vars** ou hardcoded no painel.

**Opções:**
1. **Tabela `ConfiguracoesGlobais`** com chave/valor (string) + endpoints CRUD. Flexível, simples, transversal.
2. **Tabelas dedicadas por tipo** (`TileServerConfig`, `DistritoIndustrial`, etc.). Mais estruturado mas mais boilerplate.
3. **Arquivo JSON em volume** — sem DB. Persiste mas não versiona, sem auditoria.

Recomendação: **Opção 1 (key/value)** pra configs escalares simples (tile server URL, opacidade default) + **Opção 2 dedicada pra GeoJSON de distritos** (relação 1-N, geometria, vai ter outras camadas como Rodovias/Ferrovias).

### Q4. GeoJSON upload — onde mora?

Distritos industriais, rodovias, ferrovias. Volumes potenciais: dezenas de polígonos por município, ~50 municípios na região = ~2500 features. Tamanho de arquivo: 100KB-2MB cada.

**Opções:**
1. **Coluna `jsonb` no Postgres** — funciona até ~1MB por linha, query via JSON operators. Simples.
2. **Arquivo em `UPLOADS_ROOT`** (mesmo padrão de imagens de pontos institucionais). Serve estático. Postgres só guarda o path.
3. **Object storage externo** (S3/equivalente). Adia decisão de infra (já no backlog).

Recomendação: **Opção 2** — alinhado ao padrão existente (uploads de imagem), evita load de DB pra blobs grandes.

## Fundações compartilhadas (PRs base antes das telas)

A spec menciona "clustering, virtualização, lazy loading, debounce em filtros". Algumas dessas são features novas que servem **as 3 telas**. Vale extrair para PRs próprios:

### F1. Recharts integration (5 min)

`npm install recharts`. Sem decisão arquitetural — biblioteca standalone, só consumir. Setup mínimo.

### F2. Cluster de markers no mapa principal (~1 dia)

Quando N de empresas > 200, performance degrada (testado em mecanica-hermes, padrão React Leaflet). Adicionar `react-leaflet-cluster` como camada opcional no `EmpresasFilterMapExample.tsx` antes das telas novas. Servirá as 3.

Este item **acelera a Fase 2 do decompose** do mesmo componente — vale combinar com [Plano-Decompose-EmpresasFilterMap-Fase-2](Plano-Decompose-EmpresasFilterMap-Fase-2) se for tocado.

### F3. Padronização de Filter DTOs

As 3 telas vão criar 3+ filter DTOs novos. **Aplicar [Plano-Padronizacao-Filtros-DTO](Plano-Padronizacao-Filtros-DTO) antes** das telas garante consistência desde o início. Não bloqueante mas fortemente recomendado.

### F4. Settings/Configuracoes feature (na API)

Nova feature `Features/Configuracoes/` na API com endpoints CRUD:
- `GET /api/configuracoes` — lista todas (admin)
- `GET /api/configuracoes/{chave}` — get por chave (público pra tile server URL)
- `PUT /api/configuracoes/{chave}` — set (admin)

Modelo: `Configuracao { Chave: string (PK), Valor: string, Descricao: string?, AtualizadoEm: DateTime }`.

Esse é **pré-requisito da Tela 1**, mas Telas 2 e 3 podem mockar consumo até estar pronto.

## Ordem de implementação recomendada

```
            F1 (Recharts)
                │
                ▼
       F3 (Filter DTOs std)         F4 (Configuracoes API)
                │                            │
                ├────────┬───────────────────┤
                ▼        ▼                   ▼
              Tela 2   Tela 3              Tela 1
              (paralelo)                   (admin)
                │        │                   │
                └────────┴───────────────────┘
                         │
                         ▼
                F2 (Cluster markers)
                (quando densidade exigir)
```

- **F1, F3, F4** primeiro (pré-requisitos).
- **Tela 1** entrega valor pra admin configurar o ambiente.
- **Telas 2 e 3** em paralelo após F4 (consumem settings).
- **F2 (cluster)** quando o N empresas justificar — não bloqueia entrega.

## Componentes UI compartilhados

Identificar pra extrair em **`features/relatorios/shared/`** quando aparecerem em 2+ telas:

| Componente | Onde aparece |
|---|---|
| `MapaInteligente` (wrapper sobre `EmpresasFilterMapExample` com extensão por slots/props) | Telas 1, 2, 3 — todas têm mapa central |
| `LayerControl` (toggle de camadas) | Telas 1 e 3 |
| `HeatmapDensityControl` (slider) | Telas 1 e 3 |
| `EmpresaMarkerNumerado` (badge com posição) | Tela 2 |
| `RankingTable` (tabela genérica com export CSV) | Tela 2 (empresas + CNAEs) |
| `BarraProporcional` (% horizontal) | Tela 2 (Top CNAEs) |

Não criar tudo upfront — extrair quando o segundo uso aparecer (regra "rule of three"). Documentado em [Para-Devs-Convencoes](Para-Devs-Convencoes).

## Backend: novos endpoints esperados

| Endpoint | Quem usa | Auth |
|---|---|---|
| `GET /api/configuracoes` | Tela 1 | API Key |
| `PUT /api/configuracoes/{chave}` | Tela 1 | API Key |
| `GET /api/configuracoes/{chave}` | Públicos (tile server URL no widget) | Anônimo + cache |
| `POST /api/configuracoes/distritos/upload` (multipart GeoJSON) | Tela 1 | API Key |
| `GET /api/configuracoes/distritos` (lista) | Públicos | Anônimo + cache |
| `GET /api/configuracoes/distritos/{id}` (GeoJSON content) | Públicos | Anônimo + cache |
| `GET /api/relatorios/ranking-empresas?ordenarPor=faturamento&limite=10` | Tela 2 | Anônimo + cache |
| `GET /api/relatorios/cnaes-mais-frequentes?limite=10` | Tela 2 | Anônimo + cache |
| `GET /api/empresas/{id}/analise-vizinhanca?raioMetros=5000` | Tela 3 (extensão de `/neighbors`) | Anônimo + cache |
| `POST /api/relatorios/exportar?formato=csv|pdf|excel` | Tela 2 | API Key |

Mudanças em entidades:
- `Empresa.Faturamento decimal?` (após Q1 resolvida) + migration
- Nova entidade `Configuracao` + tabela
- Nova entidade `Distrito` (ou similar, conforme Q3+Q4) + tabela

## Estratégia de testes

| Camada | Cobertura |
|---|---|
| Unit | xUnit pros services novos (relatórios, configurações, distritos) |
| Integration | Suite E2E Robot — uma nova suite por endpoint maior (`04__configuracoes`, `05__relatorios`, `06__analise-vizinhanca`) |
| UI | Smoke tests Vitest pra cada tela nova (3 testes mínimos) |
| Manual | Smoke browser obrigatório antes de cada tela — fluxos golden documentados em cada plano |

## Riscos transversais

1. **Performance dos mapas** com 3+ camadas (heatmap + markers + distritos + rotas + clusters) simultâneas. Mitigação: clustering (F2), virtualização, debounce em filtros, lazy load de GeoJSON por viewport.
2. **Custo de cálculo de faturamento** se for derivado server-side em cada request. Mitigação: pre-computar + cachear (output cache da API + invalidação em mutation).
3. **Tamanho de GeoJSON na resposta** — distrito grande pode ter 1MB+. Mitigação: serve estático (Opção 2 da Q4) com cache HTTP forte (CDN-friendly).
4. **Sincronização de Settings entre tabs/sessões** — admin muda tile server na Tela 1, widget público precisa pegar a nova URL. Mitigação: cache curto (1min) + invalidação manual via botão "Atualizar".
5. **Lock-in em Recharts** — se um dia migrar pra outra lib de chart, retrabalho. Mitigação: encapsular cada gráfico em componente próprio, contrato de props simples.

## Cross-references com outros planos

- ⏭️ [Plano-Padronizacao-Filtros-DTO](Plano-Padronizacao-Filtros-DTO) — F3 acima. Aplicar antes das telas.
- 🔁 [Plano-Decompose-EmpresasFilterMap-Fase-2](Plano-Decompose-EmpresasFilterMap-Fase-2) — Tela 3 vai mexer pesado no componente do mapa. Vale combinar com Fase 2.
- ❄️ [Plano-Google-Maps-Integration](Plano-Google-Maps-Integration) — não é dependência mas se for executado, popula massa de dados pras Telas 2/3 funcionarem com volume realista.

## Definition of Done (overview)

- [ ] Q1-Q4 respondidas pelo owner antes de iniciar qualquer tela
- [ ] F1, F3, F4 entregues
- [ ] Tela 1 entregue (admin pode configurar tile server + distritos + opacidade)
- [ ] Tela 2 entregue (ranking + CNAEs + export CSV)
- [ ] Tela 3 entregue (filtros + análise de vizinhança + KPIs; com ou sem "Fornecedor/Concorrente" conforme Q2)
- [ ] Suite E2E cobre os endpoints novos
- [ ] Smoke browser passa nas 3 telas
- [ ] Documentação no `CLAUDE.md` de cada feature nova
- [ ] Items removidos de [Melhorias-Planejadas](Melhorias-Planejadas)
