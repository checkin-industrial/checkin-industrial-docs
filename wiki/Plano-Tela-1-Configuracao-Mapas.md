# Plano: Tela 1 — Configuração de Mapas (Admin)

> **Leia primeiro**: [Plano-Relatorios-Overview](Plano-Relatorios-Overview) — define fundações compartilhadas (F1-F4) e perguntas abertas (Q1-Q4) que esta tela depende.

---

## Goal

Tela administrativa para o gestor configurar a **camada base do mapa** (tile server), gerenciar **GeoJSON de distritos industriais / rodovias / ferrovias**, e definir defaults visuais (opacidade do heatmap, paleta de cores). Sem essa tela, qualquer alteração nesses parâmetros exige deploy de código ou mudança de env var.

## Contexto

- Hoje o `EmpresasFilterMapExample` usa tile server hardcoded (CartoDB Voyager). Trocar requer code change.
- Distritos industriais foram mencionados na spec original mas **nunca implementados** — `Empresa.AreaIndustrial` (string) hoje é um campo de texto livre.
- Heatmap usa `Empresa.NumeroFuncionarios` como peso, opacidade fixa no código.

## Pré-requisitos (cross-plan)

- ✅ **F4** ([Plano-Relatorios-Overview#f4](Plano-Relatorios-Overview)) — feature `Configuracoes` na API. **Bloqueia** esta tela.
- ✅ **Q3** respondida — modelo de Settings (recomendação: key-value + tabela dedicada de `Distrito`).
- ✅ **Q4** respondida — onde mora GeoJSON (recomendação: filesystem em `UPLOADS_ROOT/distritos/`).

## Estado atual

- `Empresa` tem `AreaIndustrial: string?` (campo informativo, não-relacional).
- Tile server URL: hardcoded em [`EmpresasFilterMapExample.tsx`](https://github.com/checkin-industrial/checkin-industrial-painel/blob/main/src/features/empresas/EmpresasFilterMapExample.tsx).
- `UPLOADS_ROOT` já existe (usado por `PontosInstitucionais`). Padrão pra arquivos estáticos.

## Decisões a tomar

### D1. Granularidade de configurações ⭐ recomendação marcada

Opções:
- (a) Tela única com **tabs**: "Mapa Base", "Camadas Geográficas", "Visualização".
- (b) Tela única com **acordeões** de seções.
- (c) 3 telas separadas no menu.

**Recomendação: (a) tabs.** Configs são logicamente agrupadas, mas usuário tipicamente edita uma seção por vez. Tabs reforçam isolamento. 3 telas seria over-engineering.

### D2. Upload de GeoJSON: validação

Opções:
- (a) Aceitar qualquer GeoJSON válido (RFC 7946) sem inspeção.
- (b) Validar schema mínimo (FeatureCollection, geometry tipo Polygon/MultiPolygon).
- (c) Validar + reprojetar SRID automaticamente (assume WGS84, rejeita outros).

**Recomendação: (b).** Schema minimal protege de upload corrompido sem assumir conhecimento do admin sobre SRID. Aceitar só WGS84 (rejeitar com mensagem clara se for outro).

### D3. Versionamento de configs

Opções:
- (a) Sem histórico — `PUT` sobrescreve.
- (b) Append-only log — toda mudança gera linha nova, tela "atual" mostra a última.
- (c) `AtualizadoEm` + `AtualizadoPor` na linha (sem histórico completo).

**Recomendação: (c).** Auditoria mínima sem complexidade de log. Histórico full pode entrar como melhoria futura se precisar de undo/diff.

### D4. Distrito industrial como entidade ou só GeoJSON?

Opções:
- (a) **Só GeoJSON** — features dentro do arquivo têm `properties.nome`, `properties.tipo`. Sem tabela `Distrito`.
- (b) **Tabela `Distrito`** com FK pra `Empresa.DistritoId` (relação 1-N).
- (c) **Híbrido**: tabela `Distrito` armazena metadata + path do GeoJSON; `Empresa.AreaIndustrial` continua string livre por agora.

**Recomendação: (c).** Permite consultar/filtrar por distrito sem materializar relação 1-N agora (que seria reorg grande no schema). Empresa pode ser ligada a distrito depois via tela admin separada (ou por intersecção espacial computada).

### D5. Tile server: lista pré-definida ou URL livre?

Opções:
- (a) **URL livre** — admin cola qualquer Leaflet-compatible tile URL.
- (b) **Lista pré-definida** (CartoDB, OpenStreetMap, Esri WorldStreet, Stamen) com fallback "Custom URL".
- (c) (b) + preview ao vivo no painel.

**Recomendação: (c).** Reduz erros (URL inválida) sem perder flexibilidade. Preview previne save de URL quebrada.

## Implementação passo-a-passo

### Backend — `Features/Configuracoes/` (já criado em F4)

Após F4 entregue, esta tela adiciona:

#### 1. Endpoint upload de Distrito

```
POST /api/configuracoes/distritos
- Auth: API Key
- Body: multipart/form-data (file: GeoJSON, nome: string, tipo: enum)
- Validação: GeoJSON válido + FeatureCollection + WGS84
- Persiste:
    - DB: row em `Distritos` (Id, Nome, Tipo, ArquivoPath, AtualizadoEm)
    - Disk: file em UPLOADS_ROOT/distritos/{id}.geojson
- Returns: { id, nome, tipo, arquivoUrl }
```

#### 2. Endpoint list de Distritos

```
GET /api/configuracoes/distritos
- Auth: anônimo (camadas públicas)
- Cache: 5min
- Returns: [{ id, nome, tipo, arquivoUrl, atualizadoEm }]
```

#### 3. Endpoint download de Distrito

```
GET /api/configuracoes/distritos/{id}
- Auth: anônimo
- Cache: 1h (file estática)
- Returns: GeoJSON content (Content-Type: application/geo+json)
```

#### 4. Endpoint delete de Distrito

```
DELETE /api/configuracoes/distritos/{id}
- Auth: API Key
- Soft delete: marca `Ativo=false` + remove arquivo do disk?
```

**Decisão D6**: hard delete (libera disco) ou soft (audit trail)?
**Recomendação**: soft no DB + hard no disk (recriar requer re-upload, que é raro). Justificativa: distrito raramente é deletado por engano.

#### 5. Migration EF Core

```csharp
public class Distrito {
    public Guid Id { get; set; }
    public required string Nome { get; set; }
    public DistritoTipo Tipo { get; set; }  // Industrial | Rodovia | Ferrovia | Outro
    public required string ArquivoPath { get; set; }
    public bool? Ativo { get; set; } = true;
    public DateTime AtualizadoEm { get; set; }
    public string? AtualizadoPor { get; set; }
}
```

Migration: `AddDistritos`.

### Frontend — `features/configuracoes/`

Estrutura proposta:

```
src/features/configuracoes/
├── CLAUDE.md
├── ConfiguracoesScreen.tsx       (tela principal com tabs)
├── tabs/
│   ├── MapaBaseTab.tsx           (D5 — tile server)
│   ├── CamadasGeograficasTab.tsx (D2/D4 — upload + lista distritos)
│   └── VisualizacaoTab.tsx       (opacidade heatmap, paleta)
├── api/
│   ├── useConfiguracoes.ts       (useQuery)
│   ├── useUpdateConfiguracao.ts  (useMutation)
│   ├── useDistritos.ts
│   └── useUploadDistrito.ts
├── components/
│   ├── TileServerPreview.tsx     (mini-mapa Leaflet de teste)
│   ├── DistritoCard.tsx          (item da lista + delete)
│   └── GeoJsonValidator.tsx      (UI-side validation antes do upload)
└── ConfiguracoesScreen.test.tsx  (smoke)
```

### Fluxos UI

1. **Trocar tile server**: combo box (D5b) → preview ao vivo → "Salvar". Confirmação visual.
2. **Upload distrito**: drag-drop area + nome + select tipo → preview do polygon no mini-mapa → "Upload".
3. **Listar distritos**: cards com nome, tipo, miniatura do polygon, data de atualização, botão "Remover".
4. **Opacidade heatmap**: slider 0-100% com preview ao vivo no widget.

### Integração no menu

Adicionar em `App.tsx` ou roteamento principal:
- Item de menu "Configurações" (visível só com API Key, mesmo padrão de EmpresasManagementScreen).

## Estratégia de testes

| Camada | Cobertura |
|---|---|
| Unit (API) | `ConfiguracoesService` get/set, `DistritoService` upload/validação GeoJSON |
| E2E (Robot) | Nova suite `04__configuracoes.robot`: round-trip de configuração (set → get), upload GeoJSON válido (201), upload inválido (400), download de distrito (200 + Content-Type) |
| UI (Vitest) | Smoke da tela com mock de useQuery (3 tabs renderizam) |
| Manual | Trocar tile server → ver no mapa público; upload de distrito real → renderiza como camada na Tela 3 |

## Riscos + edge cases

1. **GeoJSON gigante** (>10MB) — limite multipart no Kestrel + mensagem clara no UI.
2. **WGS84 assumption** — admin envia EPSG:3857; sistema rejeita. Mensagem precisa instruir reprojeção (link pra ferramenta tipo `mapshaper.org`).
3. **Concorrência em settings** — dois admins editando "ao mesmo tempo". Aceitar last-write-wins (não é hot path).
4. **Tile server fora do ar** — preview retorna 404; UI mostra "Tile server inválido ou indisponível" sem salvar config.
5. **CORS de tile server externo** — Leaflet faz GET de tile, deve funcionar; só dar erro claro se navegador bloquear.
6. **Path traversal no nome do distrito** — sanitizar nome antes de usar como filename (já temos util pra isso em `PontosInstitucionais`).

## Definition of Done

- [ ] F4 entregue (pré-requisito)
- [ ] Migration `AddDistritos` aplicada
- [ ] 4 endpoints novos (`/distritos` CRUD + `/configuracoes` get/set) com testes E2E verdes
- [ ] Tela `ConfiguracoesScreen` com 3 tabs funcionais
- [ ] Preview de tile server funciona com mock de fallback
- [ ] Upload de GeoJSON com validação client-side e server-side
- [ ] Distritos uploadados renderizam como camada opcional no `EmpresasFilterMapExample` (toggle "Distritos" — F3 do Overview)
- [ ] Smoke browser: admin troca tile server, vê reflexo imediato no widget público após reload
- [ ] `CLAUDE.md` da feature `Configuracoes`
- [ ] Item movido de [Melhorias-Planejadas](Melhorias-Planejadas) pra histórico

## Estimativa

| Fase | Esforço |
|---|---|
| Backend (Configuracoes + Distritos) | 1 dia |
| Frontend (3 tabs + componentes) | 1.5 dias |
| Testes E2E + Vitest | 0.5 dia |
| Integração com mapa público (camada distritos) | 0.5 dia |
| **Total** | **~3.5 dias** |
