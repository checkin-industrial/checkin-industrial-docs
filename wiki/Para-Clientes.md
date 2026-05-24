# Para Clientes

Resumo do que a plataforma **Check-in Industrial** entrega.

## O que é

Um **widget web** embedável em sites de prefeituras, secretarias de turismo, associações comerciais ou portais regionais. Apresenta de forma visual e amigável:

- **Mapa industrial** com filtros por setor, porte, CNAE, município e situação cadastral
- **Pontos institucionais** (escolas, hotéis, ecoturismo, atrativos turísticos) em grid de cards
- **Telefones úteis** (emergência, transporte, hospedagem) agrupados por categoria
- **Análises visuais** (mapa de calor de concentração industrial, vizinhança de empresas)

A parte administrativa permite que pessoas autorizadas **cadastrem, atualizem e arquivem** todos esses registros — sem precisar saber programar.

## Casos de uso

| Cliente | Como usa |
|---|---|
| Prefeitura municipal | Mostra o parque industrial da cidade para investidores e empresas que avaliam relocação |
| Secretaria de turismo | Centraliza pontos turísticos + hotéis + telefones úteis com mapa interativo |
| Associação comercial | Mantém o cadastro setorial atualizado com os próprios membros |
| SEBRAE / SENAI regional | Cruza setor + porte + município para identificar oportunidades de atendimento |

## Benefícios sobre planilhas / sites estáticos

- **Atualização em tempo real**: admin edita um registro e o mapa público reflete imediatamente.
- **Visual e busca**: filtro por nome, setor, CNAE, município direto na URL — sem ctrl-F.
- **Geolocalização real**: cada empresa é um pino no mapa, não uma linha em planilha.
- **Histórico controlado**: soft delete preserva registros antigos para auditoria; admin pode reativar.

## Apresentação comercial

A apresentação executiva detalhada está disponível em PDF no repositório de docs:

📄 **[Apresentação Comercial da Plataforma Industrial](https://github.com/checkin-industrial/checkin-industrial-docs/blob/main/Apresentacao_Comercial_Plataforma_Industrial.pdf)** (9.7 MB)

## Próximos passos

- **Demo ao vivo**: peça acesso via canal comercial — disponibilizamos uma instância de demonstração com dados de exemplo.
- **Integração no seu site**: o painel React é embedável via iframe ou pode ser hospedado em subdomínio próprio.
- **Customização visual**: cores, logo e textos do widget público são configuráveis. Funcionalidades adicionais podem ser cotadas separadamente.
