# Plano: Integração com Google Maps

Plano técnico no formato **Plan Mode** do Claude Code. Pode ser dado direto pra outro desenvolvedor (humano ou agente) executar.

> **Status**: Adiado em 2026-05-23. Ver [Melhorias-Planejadas](Melhorias-Planejadas) para motivo e gatilho de retomada.

---

## Goal

Adicionar endpoint admin que recebe `{ cep, raioMetros, tipo }`, busca empresas via **Google Places Nearby Search** (ou alternativa OSM Overpass) no raio especificado, e cadastra cada uma no banco com `Ativo=false`. Admin revisa e reativa via painel.

Objetivo de negócio: acelerar o cadastro inicial de empresas em uma região nova, evitando trabalho manual de busca + digitação. Soft delete por default protege contra dados ruins entrando direto na produção sem revisão.

## Contexto / estado atual

- A API tem 3 features de CRUD com **soft delete consistente** (`Ativo: bool?`). Painel admin já tem botão "Reativar" pra empresas inativas — fluxo de revisão pronto.
- `Empresa.Cnpj` hoje é `[Required]` 14 dígitos com regex + unique index. Google Maps não retorna CNPJ — isso é o **gating constraint** do plano.
- A API já consome **Nominatim** (OSM) em [`Features/Geocoding/GeocodeAddress.cs`](https://github.com/checkin-industrial/checkin-industrial-api/blob/main/src/Features/Geocoding/GeocodeAddress.cs) — podemos reusar para o CEP → lat/lng.
- Painel ainda não tem UI para disparar este import. Esse é um PR separado (fora do escopo deste plano).

## Decisões a tomar

Cada item tem opções com **recomendação marcada**. Validar antes de codificar.

### D1. CNPJ obrigatório vs nullable

- ✅ **Tornar `Cnpj` nullable**. Admin preenche depois via Update antes de reativar.
- alt: rejeitar imports sem CNPJ (inviável — Google não retorna)

**Impacto**: migration que muda a coluna pra nullable + ajusta unique index pra "unique-when-not-null" (Postgres faz por default). Quebra contrato com CSV importer que assume `Cnpj` obrigatório — verificar `Importacao/ImportacaoEmpresasService.cs` antes.

### D2. Provider: Google Places ou OSM Overpass

- ✅ **Google Places Nearby Search** se billing OK
- alt: **OSM Overpass API** se billing for proibitivo

Mesma arquitetura, só muda o cliente HTTP. Recomendação depende da decisão de billing — ver "Avaliação de custos" abaixo.

### D3. Dedup de empresas re-importadas

- ✅ **`GooglePlaceId string?` único quando não-null**. Stable key fornecido pelo Google (ou `osm_id` no caso Overpass).
- alt: fuzzy match por nome+coordenada (frágil — não fazer)

**Impacto**: nova coluna `GooglePlaceId` (ou genérico `ExternalSourceId`) na tabela Empresas com unique index parcial.

### D4. Geocoding do CEP

- ✅ **Reusar Nominatim** ([`GeocodeAddress.cs`](https://github.com/checkin-industrial/checkin-industrial-api/blob/main/src/Features/Geocoding/GeocodeAddress.cs)) — gratuito e já em uso
- alt: Google Geocoding API (mais preciso, +US$ 0.005/req)

### D5. `TipoEmpresa` enum novo vs reusar `Setor`

**Pergunta aberta, não respondida na sessão original.**

- 🤔 **Opção 1**: reusar `Setor` (Industria/Comercio/Servicos). Suficiente pra começar, sem nova migration. Mapeamento Google Places types → 3 valores é "lossy" mas simples.
- 🤔 **Opção 2 (recomendação técnica)**: novo enum `TipoEmpresa` mais granular (Industria, Loja, Farmacia, Restaurante, Hotel, Posto, Banco, ...). Alinhado ao universo de tipos do Google Places. Permite filtros amigáveis no painel ("ver só farmácias").

**Decisão deve sair antes de codificar** — afeta entity, DTOs, migration, mapeamento de tipos do Google.

### D6. Síncrono vs background job

- ✅ **Síncrono no primeiro momento** — admin dispara e aguarda. Para até 60 resultados (3 páginas) deve completar em <30s.
- alt: background job (MassTransit/Hangfire) se ficar lento na prática

### D7. Limite de raio server-side

- ✅ **Máximo 10 km** — proteção contra requests caros (raio grande × resultados densos = múltiplas páginas)
- Validar inline no handler, retornar `BadRequest` se ultrapassado

## Avaliação de custos (Google Places)

Pricing atualizado em **2026-05-23** — confira <https://cloud.google.com/google-maps-platform/pricing> antes de decisão final.

| SKU | Custo unitário | O que consome |
|---|---|---|
| Places Nearby Search v1 | ~US$ 0.032 / request | 1 chamada = até 20 resultados; até 3 páginas = 60 resultados máx |
| Geocoding API (se usar Google em vez de Nominatim) | ~US$ 0.005 / request | CEP → lat/lng |
| Place Details (se enriquecer com telefone/site) | ~US$ 0.017 / request | Por empresa importada |

**Estimativa por operação (raio 10 km, ~60 resultados):**

- Nearby Search: 3 páginas × US$ 0.032 = **US$ 0.096**
- (Opcional) Place Details enriquecimento: 60 × US$ 0.017 = **US$ 1.02**
- Total: **US$ 0.10** (mínimo) a **US$ 1.13** (completo)

**Free tier** (verificar política atual):
- Modelo histórico: US$ 200 de crédito mensal automático = ~6.250 Nearby Searches grátis/mês
- Modelo novo (2024): ~10.000 chamadas de essentials grátis/mês

Para uso esperado (admin disparando esporadicamente ao mapear uma região nova), **deveria caber no free tier** — mas cartão é obrigatório.

## Implementação passo-a-passo

### Fase 1: Schema + entity

1. **Migration** `AddEmpresaCnpjNullable_AddGooglePlaceId`:
   - `Cnpj` muda pra nullable (`ALTER COLUMN Cnpj DROP NOT NULL`)
   - Unique index existente em `Cnpj` precisa virar parcial (`WHERE Cnpj IS NOT NULL`) ou similar — verificar EF gera corretamente
   - Adiciona coluna `GooglePlaceId text NULL`
   - Adiciona index único parcial `WHERE GooglePlaceId IS NOT NULL`

2. **Entity** `Empresa.cs`:
   - `Cnpj` vira `string? Cnpj { get; set; }` (sem `[Required]`)
   - Adicionar `string? GooglePlaceId { get; set; }`

3. **DTOs**:
   - `DTOEmpresaCriar` e `DTOEmpresaAtualizar`: `Cnpj` opcional
   - `DTORespostaEmpresa`: incluir `GooglePlaceId` na resposta
   - `EmpresaFilterDTO`: idem

4. **(Se D5 = Opção 2)** Adicionar enum `TipoEmpresa` + coluna + migration separada antes desta.

### Fase 2: Service de import

1. **Sub-feature** `Features/Empresas/Importacao/` — novo arquivo `ImportEmpresasFromGoogle.cs`:
   - Endpoint `POST /api/empresas/import/google-places` (admin only — `.RequireAuthorization()`)
   - Body: `DTOImportarEmpresasGoogle { Cep, RaioMetros, Tipo }`
   - Resposta: `DTOImportResult { Encontrados, Criados, JaExistentes, Ignorados[], Erros[] }`

2. **Service** `IImportEmpresasGoogleService` + impl:
   - Geocode CEP via `IGeocodingService` existente
   - Chama `IGooglePlacesClient` (novo) — Nearby Search com `location`, `radius`, `type`
   - Para cada resultado, faz `FirstOrDefault(e => e.GooglePlaceId == result.Id)` — pula se existe
   - Cria empresa com `Ativo=false` + defaults editáveis
   - Acumula contadores no `DTOImportResult`

3. **`IGooglePlacesClient`** (novo, em `Shared/HttpClients/` ou nova pasta):
   - `Task<NearbySearchResponse> NearbySearchAsync(double lat, double lng, int radius, string placeType, CancellationToken ct)`
   - Configurável via `IConfiguration["GoogleMaps:ApiKey"]`
   - Fail-fast no startup se a feature for usada sem chave configurada
   - Implementação HTTP simples (no MockServer) — `HttpClientFactory` registrado em `Program.cs`

4. **Defaults pra empresas importadas**:
   - `RazaoSocial = result.Name` (preencher com nome do Google)
   - `NomeFantasia = result.Name`
   - `MatrizOuFilial = Matriz` (default — admin ajusta)
   - `Porte = ME` (default)
   - `SituacaoCadastral = Ativa`
   - `Ativo = false` ⭐ (chave do fluxo de revisão)
   - `Setor` — mapeado do tipo (se D5 = Opção 1) ou `TipoEmpresa` setado direto (se D5 = Opção 2)
   - `Latitude`, `Longitude` — do resultado Google
   - `Endereco` — `result.Vicinity` se disponível
   - `GooglePlaceId = result.PlaceId`

5. **Mapeamento tipo → Google Places type**:
   - Tabela hardcoded `Dictionary<TipoEmpresa, string[]>`:
     - `Industria → ["industrial"]`
     - `Loja → ["store"]` (se Opção 2 da D5)
     - `Farmacia → ["pharmacy"]`
     - etc.
   - Se tipo solicitado não tiver mapeamento, retornar BadRequest com lista de tipos suportados

### Fase 3: Configuração

1. **`appsettings.json`** — adicionar seção:
   ```json
   "GoogleMaps": {
     "ApiKey": "",
     "BaseUrl": "https://places.googleapis.com/v1/"
   }
   ```

2. **`Program.cs`** — registrar client + service:
   ```csharp
   services.AddHttpClient<IGooglePlacesClient, GooglePlacesClient>();
   services.AddScoped<IImportEmpresasGoogleService, ImportEmpresasGoogleService>();
   ```

3. **Fail-fast** quando a feature for tocada sem chave configurada (não no startup geral — só se alguém chamar o endpoint). Lançar `InvalidOperationException` com mensagem clara.

### Fase 4: Testes unitários

1. **`ImportEmpresasGoogleServiceTests`**:
   - Mockando `IGooglePlacesClient` com Moq
   - Cenários: import bem-sucedido (N resultados, todos novos); dedup (1 já existe via GooglePlaceId); resposta vazia; HTTP error do Google
2. **`GooglePlacesClient`** — testar parsing da resposta JSON com fixtures (não testar HTTP real).

### Fase 5: Testes E2E (suite Robot)

Nova suite `04__google_maps_import.robot` — **MAS** depende de **mock do Google** rodando em paralelo no compose. Padrão: usar WireMock como o mecanica-hermes faz pra Mercado Pago.

- Adicionar WireMock ao `docker-compose.e2e.yml`
- Configurar API com `GoogleMaps__BaseUrl=http://wiremock:8080`
- Fixtures de mappings em `tests/resources/fixtures/wiremock/google-places-mappings/`
- Suite: dispara import, verifica empresas criadas com `Ativo=false`, dedup em segundo import

### Fase 6: Painel (PR separado, futuro)

- Tela em `Gestão → Importar do Google Maps`
- Form: CEP + radius + tipo
- Loading + tabela de resultados
- Após sucesso: redireciona pra `Empresas inativas` (filtro já existe)

## Estratégia de testes

| Camada | Cobertura | Ferramenta |
|---|---|---|
| Unit | Service com client mockado | xUnit + Moq |
| Integration | Client com WireMock | XUnit + WireMock.NET |
| E2E | Suite via stack + WireMock | Robot Framework |

## Riscos + edge cases

1. **Rate limit Google** — Nearby Search tem 1000 req/min. Improvável bater em uso admin, mas se acontecer retornar `429` com mensagem.
2. **Empresa com mesmo CNPJ que veio do Google sem CNPJ** — improvável (Google não retorna CNPJ), mas se admin re-importar após preencher CNPJ, dedup por GooglePlaceId pega.
3. **Mudança de PlaceId pelo Google** — raro mas possível. Não tem solução clean; aceitar como caso de edge.
4. **Lat/lng fora do raio em alguns resultados** — Google às vezes retorna resultados marginalmente fora. Filtrar server-side por distância exata se for crítico.
5. **Custo escalando inesperadamente** — adicionar logging por chamada com `requestId + custo estimado`. Considerar quota mensal interna (sem implementação automática neste PR).
6. **Timeout no Google** — usar `HttpClient` com timeout de 10s + retry policy via Polly (já em uso? confirmar).

## Definition of Done

- [ ] Migration aplicada (Cnpj nullable + GooglePlaceId + opcionalmente TipoEmpresa)
- [ ] Endpoint `POST /api/empresas/import/google-places` funciona contra Google API real (smoke manual com chave dev)
- [ ] Testes unitários cobrem o service com client mockado
- [ ] Suite E2E `04__google_maps_import.robot` passa com WireMock
- [ ] Documentação no `CLAUDE.md` de Empresas explicando o novo endpoint
- [ ] Item removido da página [Melhorias-Planejadas](Melhorias-Planejadas)
- [ ] PR description cita este plano + decisões finais de D1-D7

## Alternativa: OSM Overpass (mesmo plano, troca o cliente)

Se a decisão D2 cair em Overpass:

- Substitui `IGooglePlacesClient` por `IOverpassClient`
- Query Overpass QL: `nwr["landuse"="industrial"](around:RAIO,LAT,LNG);` (ou `amenity=pharmacy`, etc.)
- Dedup via `osm_id` em vez de `GooglePlaceId`
- Resto do plano permanece idêntico

Rate limit Overpass: ~1 req/s, sem custo. Pode atrasar imports densos.

---

## Pano de fundo da decisão de adiamento

Esta feature foi planejada em conversa de **2026-05-23** entre owner e Claude. Decisão de adiar veio quando o owner descobriu que Google Maps Platform exige cartão de crédito cadastrado mesmo no free tier.

A pergunta D5 (TipoEmpresa) ficou aberta — owner adiou a feature antes de responder. **Resposta esperada antes da retomada**.

Memória do raciocínio também salva em `~/.claude/projects/c--git-checkin-industrial/memory/future_google_maps_import.md` para sessões futuras de Claude que precisem do contexto.
