# Melhorias Planejadas

Backlog de features e melhorias **deliberadamente adiadas** com motivo explícito. Cada item tem o nível de planejamento já feito e o gatilho que destrava a retomada.

> **Como usar esta página**: quando precisar priorizar próximo ciclo, pondere os itens pelo gatilho ("o que precisa acontecer antes?"). Não tire daqui sem mover para um PR.

---

## 🗺️ Integração com Google Maps (import automático de empresas)

**Status**: Plano técnico completo. Adiado em **2026-05-23** por exigir billing GCP.

**Motivo de adiamento**: Google Maps Platform exige cartão de crédito cadastrado no Google Cloud, mesmo que o consumo fique 100% no free tier. O owner optou por aguardar maturidade em produção antes de tomar decisão de custos.

**Gatilho de retomada**: produto em uso real + decisão explícita sobre billing GCP **ou** decisão de seguir com alternativa zero-billing.

### Escopo desenhado

Endpoint admin que recebe `{ cep, raioMetros, tipo }`, geocodifica o CEP, busca empresas via **Google Places Nearby Search** no raio e cadastra cada uma com `Ativo=false`. Admin revisa e reativa via painel.

- Inicial: só tipo `industria` → mapeia para `place_type=industrial`
- Aberto pra crescer: `loja → store`, `farmacia → pharmacy`, `restaurante → restaurant`, `hotel → lodging`, etc.
- Dedup via `GooglePlaceId string?` (unique-when-not-null) — novo campo na entidade Empresa
- `Cnpj` precisaria virar **nullable** (Google não retorna CNPJ; admin preenche depois)

### Avaliação de custos (Google Places API)

Pricing oficial (atualizado em **2026-05-23** — confira <https://cloud.google.com/google-maps-platform/pricing> antes de tomar decisão final):

| SKU | Custo unitário | O que consome |
|---|---|---|
| Places Nearby Search v1 | **~US$ 0.032 / request** | 1 chamada = até 20 resultados; até 3 páginas = 60 resultados máximo |
| Geocoding API | **~US$ 0.005 / request** | Para converter CEP → lat/lng (alternativa: usar Nominatim grátis) |
| Place Details (se buscar telefone/site) | **~US$ 0.017 / request** | Por empresa importada |

**Estimativa por operação de import (raio 10 km, ~60 resultados):**

- Nearby Search: 3 páginas × US$ 0.032 = **US$ 0.096**
- (Opcional) Place Details para enriquecer: 60 × US$ 0.017 = **US$ 1.02**
- (Se usar Geocoding Google): + US$ 0.005

**Total por operação**: **US$ 0.10** (mínimo) a **US$ 1.13** (com enriquecimento completo).

**Free tier (verificar política atual)**:
- Modelo histórico: US$ 200 de crédito mensal automático — equivalente a ~6.250 Nearby Searches grátis/mês.
- Modelo novo (anunciado 2024): cotas por SKU. ~10.000 chamadas de essentials grátis/mês.

Para o uso esperado (admin disparando import esporádico ao mapear uma região nova), **deveria caber no free tier**. Mas há que se ter cartão cadastrado.

### Alternativa zero-billing: OpenStreetMap Overpass API

Plano técnico equivalente, troca o cliente HTTP:

- **Custo**: zero, sem cadastro, sem cartão
- **Dados**: comunitários (OSM). Qualidade boa em capitais, fraca no interior do Brasil
- **Query**: Overpass QL — pedir POIs com `landuse=industrial` ou `amenity=*` dentro de raio
- **Rate limit**: ~1 req/segundo. Aceitável pra uso admin
- Já temos **Nominatim** (mesma família OSM) em uso para geocoding — padrão técnico já no projeto

### Decisão arquitetural pendente

Quando esta feature voltar à mesa, **uma decisão precisa ser tomada antes do código**: novo enum `TipoEmpresa` (granular: Indústria/Loja/Farmácia/Hotel/...) ou reusar `Setor` existente (3 valores: Indústria/Comércio/Serviços)?

Recomendação técnica: **novo enum**, alinhado a Google Places types, separado de `Setor`. Detalhes no plano arquivado da sessão de 2026-05-23.

---

## 🔄 Decompose do `EmpresasFilterMapExample.tsx` (Fase 2)

**Status**: Fase 1 completa. Fase 2 pendente.

**Contexto**: O componente do mapa público acumulou ~1500 linhas (filtros + popup + rota OSRM + sub-mapa de vizinhança). Fase 1 extraiu helpers e migrou para `useQuery`. Fase 2 quebraria em sub-componentes: `FilterPanel`, `EmpresaPopup`, `NeighborhoodOverlay`, `RouteOverlay`, `StatsPanel`.

**Gatilho**: próxima feature que precise mexer no mapa principal — refactor + feature na mesma PR.

---

## 🎨 CSS Modules por feature

**Status**: planejado.

**Contexto**: `src/styles.css` global tem ~2890 linhas. Migrar para `<Componente>.module.css` ao lado de cada componente facilita manutenção e elimina conflitos de seletor.

**Gatilho**: feature visual nova grande, ou refactor de tema (dark mode etc.).

---

## 🧪 Smoke tests para features restantes do painel

**Status**: 3 features cobertas. 4 pendentes.

**Faltam**: `PontosInstitucionaisCardsScreen`, `TelefonesUteisManagementScreen`, `PontosInstitucionaisManagementScreen`, `EmpresasManagementScreen`.

**Estimativa**: ~30 min por feature. Padrão já estabelecido em `TelefonesUteisCardsScreen.test.tsx`.

**Gatilho**: regressão em produção numa dessas features, ou tempo livre de mantenedor.

---

## 🐳 Tecnologia de deploy

**Status**: **bloqueador para colocar em produção**.

**Contexto**: O `checkin-industrial-infra` é placeholder. Opções avaliadas mas não decididas:

- **Railway** — referenciado no `docker-compose` original. Setup mais rápido. Custo previsível.
- **Fly.io** — multi-region grátis, pode escalar pra clusters depois.
- **GCP Cloud Run / Cloud SQL** — escala automática, paga só pelo uso.
- **VPS tradicional + Docker** (Hetzner / DigitalOcean) — mais barato em volume baixo, exige mais sysadmin.

**Decisões dependentes**: SSL/domínio, backup do Postgres, monitoring (Sentry? OpenTelemetry?), CDN para painel.

**Gatilho**: primeiro cliente real / SLA definido.

---

## 🔐 Swagger SecurityDefinition pra X-Api-Key

**Status**: removido temporariamente.

**Contexto**: O upgrade pra Swashbuckle 10 mudou a API de `Microsoft.OpenApi.Models`. UI do Swagger continua funcionando, só não tem botão "Authorize" para o header `X-Api-Key`.

**Gatilho**: próximo refactor de DI da API ou quando alguém precisar testar endpoints autenticados via UI do Swagger.

---

## 📦 Mover migrations EF Core para job dedicado

**Status**: rodando no startup.

**Contexto**: `Program.cs` chama `db.Database.Migrate()` no startup do container. Funciona em **single-instance**; em multi-instance (load-balanced) há race condition.

**Gatilho**: deploy multi-instance (depende da decisão de infra).

---

## 🔁 Padronização de filtros via DTO (na API)

**Status**: parcial.

**Contexto**: Diferentes endpoints de filtro têm assinaturas diferentes (`EmpresaFilterParams`, `DTOPontoInstitucionalFiltroParams`, `DTOTelefoneUtilFiltroParams`). Cada um parseia ativo/setor/tipo de jeito ligeiramente diferente.

**Gatilho**: terceira feature nova com filtro complexo, ou se for adicionar paginação cross-feature.

---

## ✅ Como mover algo daqui pra "feito"

1. Abrir branch + PR no(s) repo(s) afetado(s).
2. PR descrição cita este item da Wiki.
3. Após merge: editar esta página, **remover** o item ou movê-lo pra uma seção "Histórico" com link para o PR que entregou.
