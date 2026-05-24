# Check-in Industrial

Plataforma web pra **mapeamento e gestão de empresas industriais** com componentes públicos (mapa, contatos úteis, pontos institucionais) e área administrativa protegida por API Key.

A plataforma é construída como **widget lightweight embarcável** em sites de prefeituras, associações comerciais ou portais regionais — sem login para o cidadão final, sem dependência de roteamento por URL, sem framework pesado.

---

## Por onde começar

| Você é... | Vá direto para |
|---|---|
| 👔 Stakeholder / cliente | **[Para Clientes](Para-Clientes)** — o que a plataforma faz, casos de uso, [apresentação comercial em PDF](https://github.com/checkin-industrial/checkin-industrial-docs/blob/main/Apresentacao_Comercial_Plataforma_Industrial.pdf) |
| 🛠️ Desenvolvedor humano | **[Arquitetura](Para-Devs-Arquitetura)** → **[Como Rodar Localmente](Para-Devs-Como-Rodar-Localmente)** → **[Convenções de Código](Para-Devs-Convencoes)** |
| 🚀 Operação / DevOps | **[CI/CD](Para-Devs-CI-CD)** — workflows, branch protection, publish Docker Hub, GitHub Pages do Allure |
| 🤖 Agente de IA (Claude Code) | **[Para IAs](Para-IAs)** — onde estão os CLAUDE.md, ordem recomendada de leitura |

---

## Visão de 30 segundos

A plataforma é composta por **5 repositórios** sob a organização [`checkin-industrial`](https://github.com/checkin-industrial) no GitHub:

```
checkin-industrial-api          .NET 10 + EF Core + Postgres        (CRUD + auth API Key)
checkin-industrial-painel       React 19 + Vite + TanStack Query    (widget + admin)
checkin-industrial-tests-e2e    Robot Framework + Docker Compose    (suite ponta-a-ponta)
checkin-industrial-docs         este repo (Wiki + apresentação)
checkin-industrial-infra        placeholder (deploy ainda não decidido)
```

Mais detalhes em [Arquitetura](Para-Devs-Arquitetura).

---

## Princípios do projeto

- **Manutenção primária por IA** — cada repo tem `CLAUDE.md` no root e em sub-pastas relevantes. Convenções são explícitas pra que agentes (e humanos novos) entrem rápido.
- **Vertical Slice Architecture** (na API) — feature = pasta. Sem camadas Domain/Application/Infrastructure separadas.
- **Soft delete consistente** — entidades nunca somem do banco; ficam com `Ativo=false` e podem ser reativadas via painel admin.
- **Sem mocks no E2E** — testes rodam contra a stack real (Postgres + .NET) via Docker Compose.
- **Auth por API Key** — sem JWT, sem OAuth. Admin recebe a chave fora-de-banda e a digita ao tentar acessar a área de gestão.
