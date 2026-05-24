# Melhorias Planejadas

Backlog de features e melhorias **deliberadamente adiadas** com motivo explícito e gatilho de retomada.

> **Como usar esta página**: quando precisar priorizar próximo ciclo, pondere os itens pelo gatilho ("o que precisa acontecer antes?"). Não tire daqui sem mover para um PR e atualizar este index.

---

## 📐 Convenção: visão humana + plano técnico

Cada melhoria pode existir em **dois formatos complementares**, deixando uma memória permanente que pode ser retomada por qualquer desenvolvedor (humano ou agente de IA):

### 1. Visão humana — esta página

Resumo de **uma seção** por item: o que é, por que foi adiado, qual o gatilho de retomada. Otimizado pra leitura rápida em reunião de priorização. Não tem detalhe técnico de implementação.

### 2. Plano técnico — página separada

Para itens com escopo bem definido, existe uma página dedicada `Plano-<nome-da-feature>` no formato **Plan Mode do Claude Code**:

- Goal + Contexto + Estado atual
- Decisões a tomar (com recomendação marcada)
- Implementação passo-a-passo (arquivos a criar/modificar)
- Estratégia de testes
- Riscos + edge cases
- Definition of done

Pode ser dado direto pra outro desenvolvedor (ou pra uma nova sessão Claude) executar com mínima orientação adicional. O plano é **memória permanente do raciocínio** — sobrevive a mudanças de equipe.

**Quem precisa de plano técnico**: itens com >2h de implementação **e** mais de uma decisão arquitetural já discutida. Itens triviais (e.g., bump de dependência) ficam só na visão humana.

### Convenção de nomeação

| Tipo de arquivo | Padrão |
|---|---|
| Índice (este arquivo) | `Melhorias-Planejadas.md` |
| Plano técnico | `Plano-<Nome-Kebab-Case>.md` |

---

## 🔄 Decompose do `EmpresasFilterMapExample.tsx` (Fase 2)

**Status**: Fase 1 completa. Fase 2 pendente.

**Motivo**: O componente do mapa público tem ~1500 linhas (filtros + popup + rota OSRM + sub-mapa de vizinhança). Fase 1 já extraiu helpers e migrou pra `useQuery`. Fase 2 quebra em sub-componentes: `FilterPanel`, `EmpresaPopup`, `NeighborhoodOverlay`, `RouteOverlay`, `StatsPanel`.

**Gatilho**: próxima feature que precise mexer no mapa principal — refactor + feature na mesma PR (justifica o investimento).

📐 **Plano técnico**: [Plano-Decompose-EmpresasFilterMap-Fase-2](Plano-Decompose-EmpresasFilterMap-Fase-2) — extração em 8 sub-fases, decisão por Context vs prop drilling, 5 smoke tests, definition of done com PR por fase.

---

## 🎨 CSS Modules por feature

**Status**: planejado.

**Motivo**: `src/styles.css` global tem ~2890 linhas. Migrar para `<Componente>.module.css` ao lado de cada componente facilita manutenção e elimina conflitos de seletor.

**Gatilho**: feature visual nova grande, ou refactor de tema (dark mode etc.).

📐 Sem plano técnico ainda — implementação é padrão (extração mecânica), não tem decisões não-óbvias.

---

## 🐳 Tecnologia de deploy

**Status**: **bloqueador para colocar em produção**.

**Motivo de adiamento**: decisão de produto, não técnica. Opções avaliadas mas não escolhidas:

- **Railway** — referenciado no `docker-compose` original. Setup mais rápido. Custo previsível.
- **Fly.io** — multi-region grátis, escala pra clusters depois.
- **GCP Cloud Run / Cloud SQL** — escala automática, paga só pelo uso.
- **VPS tradicional + Docker** (Hetzner / DigitalOcean) — mais barato em volume baixo, exige mais sysadmin.

**Decisões dependentes**: SSL/domínio, backup do Postgres, monitoring (Sentry? OpenTelemetry?), CDN para painel.

**Gatilho**: primeiro cliente real ou SLA definido.

📐 Sem plano técnico — precisa de decisão de produto antes. Quando a tecnologia for escolhida, cada uma justifica um plano próprio (Terraform pra GCP é diferente de Dockerfile + scripts pra VPS).

---

## 📦 Mover migrations EF Core para job dedicado

**Status**: rodando no startup.

**Motivo**: `Program.cs` chama `db.Database.Migrate()` no startup do container. Funciona em **single-instance**; em multi-instance (load-balanced) há race condition.

**Gatilho**: deploy multi-instance (depende da decisão de infra).

📐 Sem plano técnico — depende de qual tecnologia de deploy for escolhida (init container k8s? job CI? script de pre-deploy?).

---

## 📊 Relatórios gerenciais (3 telas)

**Status**: Planos técnicos completos. Adiado até decisão sobre Q1 (faturamento) e Q3/Q4 (modelo de Settings/GeoJSON).

**Motivo**: A spec original do produto previa três telas de relatório gerencial (configuração de mapas admin, ranking de empresas com Top 10 + CNAEs, dashboard geoespacial com análise de vizinhança). Não foram implementadas na MVP por exigirem decisões de produto ainda não tomadas (fonte de faturamento, modelagem de distritos, relação fornecedor/concorrente).

**Gatilho de retomada**: Q1 (faturamento) respondida + Q3/Q4 (Settings/GeoJSON) respondidas. Cada tela tem pré-requisitos próprios documentados no plano.

**O que faz**:

- **Tela 1** (admin): configurar tile server, upload de GeoJSON de distritos, opacidade do heatmap.
- **Tela 2** (gerencial): Top 10 empresas por faturamento (mapa numerado + tabela + export CSV) + Top CNAEs mais frequentes (barras proporcionais).
- **Tela 3** (analista): filtros avançados (porte/setor/situação/faturamento) + camadas (distritos/heatmap/rodovias) + painel de análise de vizinhança com KPIs.

📐 **Plano overview**: [Plano-Relatorios-Overview](Plano-Relatorios-Overview) — reconciliação spec ↔ stack, 4 perguntas abertas (Q1-Q4), 4 fundações compartilhadas (F1-F4), ordem de implementação.

📐 **Planos por tela**:

- [Plano-Tela-1-Configuracao-Mapas](Plano-Tela-1-Configuracao-Mapas) — admin GIS (~3.5 dias)
- [Plano-Tela-2-Ranking-Empresas](Plano-Tela-2-Ranking-Empresas) — Top 10 + CNAEs (~2.75 dias)
- [Plano-Tela-3-Dashboard-Vizinhanca](Plano-Tela-3-Dashboard-Vizinhanca) — exploração interativa (~5.5 dias sem Decompose; ~8-9 com Decompose Fase 2 antes)

---

## 🔁 Padronização de filtros via DTO (na API)

**Status**: parcial.

**Motivo**: diferentes endpoints de filtro têm assinaturas diferentes (`EmpresaFilterParams`, `DTOPontoInstitucionalFiltroParams`, `DTOTelefoneUtilFiltroParams`). Cada um parseia ativo/setor/tipo de jeito ligeiramente diferente.

**Gatilho**: o 3º caso já existe (TelefoneUtil). Retomar quando o 4º caso surgir — aí o padrão emergente fica claro o suficiente para padronizar com confiança. Padronizar agora seria preventivo demais e poderia gerar rework se o 4º caso trouxer requisito novo (paginação, ordering).

📐 **Plano técnico**: [Plano-Padronizacao-Filtros-DTO](Plano-Padronizacao-Filtros-DTO) — 6 decisões (D1-D6) com recomendação marcada, migração em 5 fases, foco em helper compartilhado em `Shared/Filters/` em vez de herança (alinhado com VSA), padronização do tratamento de `Ativo` (`string?` com helper único).

---

## ✅ Como mover algo daqui pra "feito"

1. Abrir branch + PR no(s) repo(s) afetado(s).
2. PR descrição cita este item da Wiki (ou o link pro `Plano-*` correspondente).
3. Após merge: editar esta página — **remover** o item ou movê-lo pra uma seção "Histórico" com link pro PR que entregou. Se havia `Plano-*`, deletar também (ou arquivar com nota "executado em PR #X").

---

## 📜 Histórico (entregues)

| Data | Item | PR(s) |
|---|---|---|
| 2026-05-24 | Google Maps Import (backend) | [api#13](https://github.com/checkin-industrial/checkin-industrial-api/pull/13), [api#14](https://github.com/checkin-industrial/checkin-industrial-api/pull/14) (defaults conservadores) |
| 2026-05-24 | Cluster automático de markers (>200 visíveis) | [painel#18](https://github.com/checkin-industrial/checkin-industrial-painel/pull/18) |
| 2026-05-24 | Swagger SecurityDefinition X-Api-Key | [api#12](https://github.com/checkin-industrial/checkin-industrial-api/pull/12) |
| 2026-05-24 | Smoke tests das 4 telas restantes | [painel#17](https://github.com/checkin-industrial/checkin-industrial-painel/pull/17) |
