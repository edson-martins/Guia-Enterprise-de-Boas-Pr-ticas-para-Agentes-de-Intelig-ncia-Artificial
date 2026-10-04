# Workflows n8n do Guia Enterprise de Agentes de IA

Repositório de recursos do livro **Guia Enterprise de Boas Práticas para Agentes de Inteligência Artificial: da automação ao raciocínio autônomo: arquitetura, design, segurança e escala com n8n** (Edson Martins, 1. ed., 2026).

Aqui estão os workflows citados no livro, em formato JSON pronto para importar no n8n. Cada arquivo corresponde a um fluxo descrito em um capítulo, e a tabela abaixo indica onde ele é explicado.

## Workflows disponíveis

| Workflow | Descrição | Capítulo | Nível |
|---|---|---|---|
| `atendimento-smb-base.json` | Agente de atendimento base com WhatsApp + Claude | Cap. 9 | Iniciante |
| `qualificacao-leads.json` | Qualificação de leads com 5 campos e CRM | Cap. 9 | Iniciante |
| `lembrete-agendamento.json` | Agente de lembretes com Google Calendar | Cap. 10 | Iniciante |
| `faq-dinamico-sheets.json` | FAQ com sincronização do Google Sheets (15 min) | Cap. 10 | Intermediário |
| `handoff-humano-redis.json` | Handoff para humano com TTL, retorno automático e comando `!encerrar` | Cap. 10 | Intermediário |
| `freebusy-cache.json` | Cache de disponibilidade de calendário com Redis | Cap. 10 | Intermediário |
| `error-handler-backoff.json` | Tratamento de erros com retry e alertas | Cap. 14 | Intermediário |
| `multi-agent-manager.json` | Padrão Manager + Workers especializados | Cap. 7 | Avançado |
| `avaliacao-pipeline.json` | Pipeline de avaliação automatizada de qualidade | Cap. 15 | Avançado |
| `documento-analise.json` | Extração estruturada de contratos e PDFs | Apêndice A | Avançado |

## Como importar

1. Faça o download do repositório completo (botão **Code → Download ZIP**) ou de um arquivo `.json` individual.
2. No n8n, crie um workflow novo.
3. Clique no menu **⋯** (canto superior direito) e escolha **Import from File**.
4. Selecione o arquivo `.json` correspondente.
5. Configure as credenciais de cada nó (veja a seção seguinte) e salve.

Para preparar um ambiente de estudo local antes de importar, siga o [GUIA_AMBIENTE_ESTUDO.md](GUIA_AMBIENTE_ESTUDO.md).

## Credenciais

O n8n não exporta senhas nem chaves de API. Depois de importar, cada nó que acessa um serviço externo aparece com a credencial pendente, e você precisa criar a sua. Conforme o workflow, serão necessárias credenciais de:

- **Modelo de linguagem** (Anthropic ou outro provedor compatível com o nó AI Agent)
- **WhatsApp Cloud API** (Meta)
- **Redis** (estado de sessão, handoff e cache)
- **Postgres / Supabase** (histórico e dados do agente)
- **Google Sheets** e **Google Calendar**

Substitua também as URLs de webhook e os identificadores de exemplo (planilhas, calendários, números de telefone) pelos do seu ambiente.

## Antes de usar em produção

Os workflows são templates de partida, não soluções plug-and-play. Revise todas as credenciais, substitua as URLs de webhook, ajuste os prompts de sistema para o seu contexto e execute testes extensivos em ambiente de desenvolvimento antes de expor ao usuário final.

Os capítulos 11 (Guardrails), 12 (Privacidade e LGPD) e 14 (Resiliência) do livro tratam do que precisa estar resolvido antes de um agente atender clientes reais.

## Relação com o livro

No livro, os fluxos aparecem em notação YAML, uma forma didática de mostrar a lógica de cada workflow nó a nó. Os arquivos deste repositório são a versão importável desses mesmos fluxos.

## Contribuindo

Adaptou algum workflow para o seu contexto e quer compartilhar? A [comunidade n8n](https://community.n8n.io/) é o espaço ideal. Use a tag **#AgentesIA** para facilitar a descoberta por outros praticantes brasileiros.

Encontrou um erro em algum workflow? Abra uma *issue* neste repositório.

## Uso e licença

Os workflows acompanham o livro e seguem as condições de uso descritas no Apêndice C. A redistribuição pública, a venda ou a criação de produtos derivados para comercialização dependem de autorização expressa por escrito do autor.

## Como citar

> MARTINS, Edson. **Guia Enterprise de Boas Práticas para Agentes de Inteligência Artificial**: da automação ao raciocínio autônomo: arquitetura, design, segurança e escala com n8n. 1. ed. São Paulo: Edição do autor, 2026.

## Contato

- E-mail: contato@edsonmartins.com
- LinkedIn: [linkedin.com/in/edson-martins](https://linkedin.com/in/edson-martins)
