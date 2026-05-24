# Check-in Industrial — Documentação

Repositório de **documentação central** do ecossistema [Check-in Industrial](https://github.com/checkin-industrial). É a fonte de verdade para o Wiki público do GitHub e para a apresentação comercial.

📖 **Wiki publicado**: <https://github.com/checkin-industrial/checkin-industrial-docs/wiki>

📄 **Apresentação Comercial** (PDF): [`Apresentacao_Comercial_Plataforma_Industrial.pdf`](./Apresentacao_Comercial_Plataforma_Industrial.pdf)

## Estrutura

```text
checkin-industrial-docs/
├── README.md                                           (este arquivo)
├── Apresentacao_Comercial_Plataforma_Industrial.pdf    (deck comercial — 9.7 MB)
├── wiki/                                               (source-of-truth do Wiki)
│   ├── Home.md
│   ├── Melhorias-Planejadas.md                         (index de backlog — visão humana)
│   ├── Plano-Google-Maps-Integration.md                (plano técnico Plan Mode — exemplo canônico)
│   ├── Para-Clientes.md
│   ├── Para-Devs-Arquitetura.md
│   ├── Para-Devs-Como-Rodar-Localmente.md
│   ├── Para-Devs-Convencoes.md
│   ├── Para-Devs-CI-CD.md
│   ├── Para-IAs.md
│   ├── _Sidebar.md
│   └── _Footer.md
└── .github/workflows/
    └── publish-wiki.yml                                (sync wiki/ → GitHub Wiki em push main)
```

## Como editar o Wiki

**Não edite pela UI do GitHub Wiki.** Em vez disso:

1. Faça as mudanças em `wiki/*.md`
2. Abra um PR neste repositório
3. Após merge em `main`, o workflow [`publish-wiki.yml`](.github/workflows/publish-wiki.yml) sincroniza automaticamente o conteúdo para o Wiki nativo

Vantagem: edits passam por code review como qualquer mudança de código.

## Setup inicial (one-time)

O GitHub Wiki precisa de **uma página criada pela UI** antes do workflow tomar conta:

1. Settings → Features → confirme que **Wikis** está habilitado
2. Aba **Wiki** → "Create the first page" → conteúdo placeholder qualquer → Save
3. Próximo push em `main` na pasta `wiki/` dispara o workflow que sobrescreve com o conteúdo correto

Após o setup inicial, todas as edições futuras passam pelo PR + workflow.

## Convenções de markdown

- Nomes de arquivo em `wiki/`: `Kebab-Case-With-Hyphens.md`. GitHub Wiki os interpreta como espaços nas URLs.
- Links entre páginas do Wiki: `[Nome da Página](Nome-Da-Pagina)` (sem `.md`, sem path)
- Links para código: usar URL absoluta do GitHub (`https://github.com/checkin-industrial/.../blob/main/...`) para que o link funcione tanto no Wiki quanto em qualquer outro contexto
- Arquivos com prefixo `_` (sublinhado) são especiais do GitHub Wiki:
  - `_Sidebar.md` — sidebar de navegação
  - `_Footer.md` — footer

## Repositórios do ecossistema

| Repo | Papel |
|---|---|
| [`checkin-industrial-api`](https://github.com/checkin-industrial/checkin-industrial-api) | Backend .NET 10 |
| [`checkin-industrial-painel`](https://github.com/checkin-industrial/checkin-industrial-painel) | Frontend React 19 |
| [`checkin-industrial-tests-e2e`](https://github.com/checkin-industrial/checkin-industrial-tests-e2e) | Suite E2E Robot Framework |
| **`checkin-industrial-docs`** | **Este repo** — Wiki + apresentação |
| [`checkin-industrial-infra`](https://github.com/checkin-industrial/checkin-industrial-infra) | Infra (placeholder — deploy pendente) |
