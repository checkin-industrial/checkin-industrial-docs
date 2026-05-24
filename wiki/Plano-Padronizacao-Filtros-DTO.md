# Plano: Padronização dos Filter DTOs da API

Plano técnico no formato **Plan Mode** do Claude Code. Pode ser dado direto pra outro desenvolvedor (humano ou agente) executar.

> **Status**: ✅ **Entregue em 2026-05-24** ([api#16](https://github.com/checkin-industrial/checkin-industrial-api/pull/16)). Empresa.Status fica com parsing próprio (3 estados, domain-specific); Ponto/Telefone migrados para `Ativo: string?` com helper compartilhado em `src/Shared/Filters/FilterHelpers.cs`. Plano preservado como referência da arquitetura adotada (D1-D6 todas executadas).

---

## Goal

Padronizar a forma como **filter DTOs** são modelados, parseados e aplicados nos endpoints `GET /api/<recurso>/filter` (ou `?` direto na listagem). Hoje cada feature tem assinatura ligeiramente diferente — o que dificulta adicionar filtros novos, paginação, e leva a bugs sutis (e.g., tratamento de `ativo` divergente entre features).

Objetivo final: cada feature herda uma base ou usa helpers comuns, mantém só os filtros específicos da feature, e o tratamento de filtros transversais (`ativo`, paginação futura) é único.

## Contexto / estado atual

3 filter DTOs hoje, todos públicos em `Features/<X>/`:

### `EmpresaFilterParams`
```csharp
public string? NomeFantasia { get; set; }
public string? Setor { get; set; }
public string? Porte { get; set; }
public string? Cnae { get; set; }
public string? Municipio { get; set; }
public string? Situacao { get; set; }
public int? MinFuncionarios { get; set; }
public int? MaxFuncionarios { get; set; }
public string? Ativo { get; set; }  // ⚠️ string "true"/"false" parseado manualmente
```

### `DTOPontoInstitucionalFiltroParams`
```csharp
public string? Tipo { get; set; }
public bool? Ativo { get; set; }    // ⚠️ bool? — ASP.NET model binding nativo
```

### `DTOTelefoneUtilFiltroParams`
```csharp
public string? Categoria { get; set; }
public bool? Ativo { get; set; }    // ⚠️ bool? — igual Ponto
public string? Termo { get; set; }
```

### Divergências observadas

1. **`Ativo`**: `string?` em Empresa, `bool?` em Telefone/Ponto. **Inconsistência real**.
   - `string?` permite passar `?ativo=todos` (sem efeito), `?ativo=true`, `?ativo=false`. Parsing manual em `EmpresaFilterQuery.ParseAtivo`.
   - `bool?` aceita só `?ativo=true` / `?ativo=false`. Ausente = `null` = "todos". Mais idiomático ASP.NET.
2. **Naming dos enums-string**: Empresa parseia `setor` como `"industria"`/`"comercio"`/`"servicos"`; Ponto parseia `tipo` com aliases (`"servico"` E `"servicos"`); Telefone parseia `categoria` com formato próprio. Cada `ParseX` é re-implementado por feature.
3. **Filtros transversais não-existentes**: nenhum DTO tem `Page`, `PageSize`, `OrderBy` — quando paginação chegar, será adicionada de jeito ad-hoc se não houver padrão.
4. **Sem nullability convention**: todos os campos são nullable (✓ correto pra filtros opcionais), mas alguns features fazem `string.IsNullOrWhiteSpace`, outros fazem `!= null` — micro-divergência.

## Decisões a tomar

### D1. Forma do tratamento de `ativo` (escolher UM)

- 🤔 **Opção 1**: padronizar em `bool?` (como Telefone/Ponto). ASP.NET binding faz o trabalho. Empresa muda.
- ✅ **Opção 2 (recomendação)**: padronizar em `string?` com helper `ParseAtivo` compartilhado. Aceita "true"/"false"/"todos"/null. Mais flexível pra UI (painel pode mandar `"todos"` explicitamente em vez de ausente). Telefone/Ponto migram.

Justificativa pra Opção 2: o painel já tem dropdown "Somente ativas | Somente inativas | Todas" — "Todas" virar `?ativo=todos` é mais explícito que "omitir parâmetro". Ajuda debug e logs.

### D2. Escopo: padronização-only vs padronização + paginação

- ✅ **Padronização-only neste PR** — adicionar paginação cria muito acoplamento (clientes, ordering, total count, etc.) e é fácil de retroativar depois.
- alt: bundle paginação (risco grande, escopo enorme)

Quando paginação for necessária (presumivelmente após >10k registros em produção), aí sim entra como follow-up dedicado.

### D3. Forma técnica da padronização (heranças)

- 🤔 **Opção A**: classe base abstrata `FilterParamsBase` com `string? Ativo`. Cada DTO herda.
- ✅ **Opção B (recomendação)**: **helpers estáticos** em `Shared/Filters/FilterHelpers.cs`. Cada DTO mantém seu próprio shape (sem herança) mas usa `FilterHelpers.ParseAtivo(filtros.Ativo)` no service/query.
- alt: interface `IHasAtivoFilter` (zero benefício extra)

Justificativa pra Opção B: VSA enfatiza isolamento por feature. Herança cria acoplamento "shared types" que VSA quer evitar. Helper é uma função pura — fácil de testar, fácil de mockar, sem inverter dependências.

### D4. Migration dos parsers existentes

- ✅ **Mover funções `ParseXxx`** que são realmente compartilhadas (`ParseAtivo`) para `Shared/Filters/FilterHelpers.cs`. Manter `ParseSetor`/`ParsePorte`/`ParseSituacao` em `EmpresaFilterQuery` (são domain-specific).
- Documentar no `Shared/Filters/CLAUDE.md` (novo) o critério: "é compartilhado se aparecer em ≥2 features ou se for um filtro transversal claro".

### D5. Naming convention dos DTOs

- 🤔 **Opção 1**: padronizar nome — todos viram `<Feature>FilterParams` (Empresa já assim; Telefone/Ponto mudam de `DTO<Feature>FiltroParams` pra `<Feature>FilterParams`)
- 🤔 **Opção 2 (recomendação)**: **manter como está**. Renomear classes públicas é breaking change interno; o ganho é puramente cosmético. Documentar o padrão pra DTOs **novos** (escolher um), mas não renomear os existentes.

### D6. Quando aplicar este plano

- ✅ **Quando o 4º caso de filter DTO surgir** (3º já chegou em [`DTOTelefoneUtilFiltroParams`](https://github.com/checkin-industrial/checkin-industrial-api/blob/main/src/Features/TelefonesUteis/DTOTelefoneUtilFiltroParams.cs)). A regra heurística "esperar o 3º caso" foi premissa do owner mas o 3º já existe — o que falta é o padrão emergente ficar claro, o que vai acontecer com o 4º.
- alt: aplicar agora (preventivo) — desconsiderado. Sem o 4º caso, padrão pode estar errado e gerar refactor duplicado.

## Implementação passo-a-passo

### Fase 1: Helper compartilhado (~30min)

1. Criar `src/Shared/Filters/FilterHelpers.cs`:
   ```csharp
   namespace AppTurismoIndustrial.Api.Shared.Filters;

   public static class FilterHelpers
   {
       public static bool? ParseAtivo(string? value)
       {
           if (string.IsNullOrWhiteSpace(value)) return null;
           return value.Trim().ToLowerInvariant() switch
           {
               "true" => true,
               "false" => false,
               "todos" => null,
               _ => null
           };
       }
   }
   ```
2. Criar `src/Shared/Filters/CLAUDE.md` documentando critério "é compartilhado se ≥2 features usam OU se for transversal".
3. Adicionar `Shared.Filters` aos global usings em `AppTurismoIndustrial.Api.csproj` + `AppTurismoIndustrial.Api.Tests.csproj`.

### Fase 2: Migrar `EmpresaFilterQuery` para usar o helper (~15min)

1. Em [`EmpresaFilterQuery.cs`](https://github.com/checkin-industrial/checkin-industrial-api/blob/main/src/Features/Empresas/EmpresaFilterQuery.cs), substituir o `ParseAtivo` local pelo `FilterHelpers.ParseAtivo`.
2. Deletar o método `ParseAtivo` privado.
3. Testes unitários permanecem (cobrem comportamento, não a função em si).

### Fase 3: Padronizar Telefone e Ponto (~1h)

Esta é a parte com **impacto na API pública** — mudar `Ativo` de `bool?` pra `string?`.

1. `DTOPontoInstitucionalFiltroParams`:
   ```csharp
   public string? Tipo { get; set; }
   public string? Ativo { get; set; }   // era bool?
   ```
2. `DTOTelefoneUtilFiltroParams`: idem.
3. Atualizar `PontoInstitucionalQuery.ConsultarAsync` e `TelefoneUtilQuery.ConsultarAsync` para usar `FilterHelpers.ParseAtivo(filtros.Ativo)`.
4. **Backward compat**: ASP.NET model binding aceita `?ativo=true` em ambos os casos (string e bool?). Mudança fica transparente pra clientes existentes — o que muda é que agora `?ativo=todos` passa a ser válido.
5. Atualizar painel: nas chamadas que usavam `?ativo=true|false`, **nada muda**. Adicionar `?ativo=todos` explícito onde antes era "omitir o param" se quiser explicitude (opcional).
6. Atualizar suite E2E ([`telefones_api.resource`](https://github.com/checkin-industrial/checkin-industrial-tests-e2e/blob/main/tests/resources/keywords/telefones_api.resource), [`pontos_api.resource`](https://github.com/checkin-industrial/checkin-industrial-tests-e2e/blob/main/tests/resources/keywords/pontos_api.resource)) — verificar que keywords não dependem do tipo específico do query param.

### Fase 4: Testes unitários do helper (~30min)

1. `src/tests/AppTurismoIndustrial.Api.Tests/Shared/Filters/FilterHelpersTests.cs`:
   - `ParseAtivo` retorna `true` pra `"true"`, `"True"`, `"TRUE"`
   - Retorna `false` pra `"false"`, etc.
   - Retorna `null` pra `null`, `""`, `"   "`, `"todos"`, `"qualquer-coisa-invalida"`

### Fase 5: Validação (~15min)

1. `dotnet build` + `dotnet test` (esperado: 22 + N novos testes)
2. `docker build .` + subir stack E2E + rodar suite Robot
3. Pelo lado do painel: `npm run typecheck` + `npm test` (nada deve quebrar — comportamento é compatível)

## Estratégia de testes

| Camada | Cobertura | Ferramenta |
|---|---|---|
| Unit | `FilterHelpersTests` cobre os casos do `ParseAtivo` | xUnit |
| Integration | `EmpresaFilterQuery` + `PontoInstitucionalQuery` + `TelefoneUtilQuery` continuam com seus testes existentes — comportamento não muda | xUnit |
| E2E | Suite Robot continua verde — query string `?ativo=true` continua funcionando como antes | Robot |

## Riscos + edge cases

1. **Quebra de cliente externo**: se algum integrador batia com `?ativo=1` (1 = truthy em alguns parsers), a mudança pra `string?` rejeita silenciosamente. **Mitigação**: aceitar `"1"`/`"0"` no `ParseAtivo` se algum cliente real usar. Por padrão, não — restritivo é mais previsível.
2. **Cache de OpenAPI/Swagger schema**: o tipo de `Ativo` muda no JSON Schema do Swagger (de `boolean` pra `string`). Não impacta runtime, mas atualiza geração automática de client. **Mitigação**: nota no PR + bump de versão minor.
3. **Front-end já mandando `?ativo=true|false` literal**: sem impacto, continua funcionando.
4. **Painel mandando "?ativo=todos"**: antes não era reconhecido (caía no fallback `null`). Agora explicitamente vira `null`. Comportamento idêntico do ponto de vista do servidor, mas semanticamente mais correto.

## Definition of Done

- [ ] `Shared/Filters/FilterHelpers.cs` criado com `ParseAtivo` + testes
- [ ] `Shared/Filters/CLAUDE.md` criado com critério de "compartilhável"
- [ ] 3 features (`Empresa`, `Ponto`, `Telefone`) usam `FilterHelpers.ParseAtivo`
- [ ] 3 DTOs com `Ativo` como `string?`
- [ ] Suite unitária + E2E verde
- [ ] Painel (que já manda `?ativo=true|false`) continua funcionando — confirmar via smoke browser
- [ ] Item removido de [Melhorias-Planejadas](Melhorias-Planejadas)
- [ ] PR descrição lista decisões D1-D6 e qualquer divergência

## Pano de fundo da decisão de adiamento

Adiado em 2026-05-23 com a regra "esperar o 3º caso". O **3º caso já existe** (TelefoneUtil), mas o owner manteve a feature adiada por estar focando em outras prioridades. Esse plano fica em pé pra ser executado quando o 4º caso surgir (forte indicador de que o padrão emergente é claro o suficiente).

**Riscos de executar cedo demais**: aplicar uma abstração antes do padrão emergente ficar claro pode resultar em rework. Os 3 casos atuais convergem em `ativo` mas divergem em formato (`string?` vs `bool?`) — esse plano resolve essa divergência específica, mas se o 4º caso trouxer um requisito novo (e.g., paginação, ordering), vale revisitar antes de mergulhar.
