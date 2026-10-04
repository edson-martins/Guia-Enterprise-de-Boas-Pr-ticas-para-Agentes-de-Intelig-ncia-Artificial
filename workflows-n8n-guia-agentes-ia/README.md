# Biblioteca de Workflows — Guia Enterprise de Boas Práticas para Agentes de IA

Workflows referenciados no Apêndice B do livro. Todos usam nodes nativos
do n8n (HTTP Request, Postgres, Redis) e os nodes de IA do pacote
`@n8n/n8n-nodes-langchain` (AI Agent, Chat Model, Memory, Tools) para os
fluxos que envolvem raciocínio do agente — consistente com o que o
Capítulo 9.3 ensina.

**Este é um ambiente de estudo isolado, independente de qualquer sistema
em produção.** As tabelas (`business`, `faq`, `appointments`, etc.) e a
API de apoio (`ambiente-estudo-api.json`) foram criadas só pra você
conseguir rodar os exemplos do livro do zero, com dados de teste — não
representam o schema de nenhum produto real. **Comece pelo
`GUIA_AMBIENTE_ESTUDO.md`**, que tem o passo a passo completo de setup
antes de importar qualquer workflow.

**Estes são templates de partida, não soluções plug-and-play** (mesmo
aviso do Apêndice B): revise credenciais, URLs de webhook e prompts de
sistema antes de qualquer uso em produção.

## Como importar

No painel do n8n: menu lateral → **Workflows** → **Import from File** →
selecione o `.json` correspondente.

## Tabela de referência

| Arquivo | Descrição | Capítulo | Nível |
|---|---|---|---|
| `ambiente-estudo-api.json` | API de apoio (companion) — endpoints de estudo backed por Postgres | — | Setup |
| `atendimento-smb-base.json` | Agente de atendimento base com WhatsApp + Claude | Cap. 9 | Iniciante |
| `qualificacao-leads.json` | Qualificação de leads com 5 campos e CRM | Cap. 9 | Iniciante |
| `faq-dinamico-sheets.json` | FAQ com sync Google Sheets (15 min) | Cap. 10 | Intermediário |
| `handoff-humano-redis.json` | Handoff com TTL auto-return e comando `!encerrar` | Cap. 10 | Intermediário |
| `multi-agent-manager.json` | Padrão Manager + Workers especializados | Cap. 7 | Avançado |
| `sub-agente-triagem-exemplo.json` | Sub-agente de triagem (bônus, não listado no livro) | Cap. 7 | Avançado |
| `sub-agente-faq-exemplo.json` | Sub-agente de FAQ (bônus, não listado no livro) | Cap. 7 | Avançado |
| `sub-agente-agendamento-exemplo.json` | Sub-agente de agendamento (bônus, não listado no livro) | Cap. 7 | Avançado |
| `sub-agente-suporte-exemplo.json` | Sub-agente de suporte técnico (bônus, não listado no livro) | Cap. 7 | Avançado |
| `error-handler-backoff.json` | Tratamento de erros com retry e alertas | Cap. 14 | Intermediário |
| `avaliacao-pipeline.json` | Pipeline de avaliação automatizada de qualidade | Cap. 15 | Avançado |
| `lembrete-agendamento.json` | Agente de lembretes com Google Calendar | Cap. 10 | Iniciante |
| `freebusy-cache.json` | Cache de disponibilidade de calendário com Redis | Cap. 10 | Intermediário |
| `documento-analise.json` | Extração estruturada de contratos e PDFs | Apênd. A | Avançado |

## Setup completo (schema, API de estudo, variáveis, ordem de importação)

Está tudo em **`GUIA_AMBIENTE_ESTUDO.md`** — schema SQL com dados de
exemplo (`schema-estudo.sql`), a API de apoio, a tabela completa de
variáveis de ambiente, credenciais necessárias, e a ordem correta de
importação e teste. Esta seção do README só resume o essencial; o guia
é a fonte de verdade.

## Credenciais a configurar no n8n

- **Anthropic** (nos nodes `Anthropic Chat Model`, tipo `@n8n/n8n-nodes-langchain.lmChatAnthropic`)
- **Postgres** (em todos os nodes Postgres) — aponte pro banco criado com `schema-estudo.sql`
- **Redis** (workflows 04 e 09)
- **Google Sheets OAuth2** (workflow 03, opcional)
- **Google Calendar OAuth2** (workflow 09, opcional)

## Sobre o padrão Multi-Agente (workflow 05)

O `multi-agent-manager.json` referencia 4 sub-workflows por ID
(`SUBSTITUA_PELO_ID_...`). Os 4 já estão nesta biblioteca prontos pra
importar (`sub-agente-triagem-exemplo.json`, `sub-agente-faq-exemplo.json`,
`sub-agente-agendamento-exemplo.json`, `sub-agente-suporte-exemplo.json`)
— só falta importar cada um, copiar o ID do workflow que o n8n gera, e
colar no campo `workflowId` do node `Tool` correspondente no Manager.

Dependências entre eles:
- **Sub-agente de FAQ** precisa do workflow 03 (`faq-dinamico-sheets.json`) importado e ativo — consome o webhook `/faq-consultar` dele.
- **Sub-agente de Agendamento** precisa do workflow 09 (`freebusy-cache.json`) importado e ativo, e de rotas `/appointments` e `/appointments/cancel` na sua API interna.
- **Sub-agente de Suporte Técnico** precisa do workflow 04 (`handoff-humano-redis.json`) importado e ativo — consome o webhook `/handoff-ativar` dele.
- **Sub-agente de Triagem** não tem dependência externa — só a chamada ao modelo.

Ordem recomendada de importação: 03, 04, 09 → depois os 4 sub-agentes →
depois o `multi-agent-manager.json` por último (é o único que referencia
os IDs dos outros).

## Dúvidas ou problemas com um fluxo específico

Descreva o comportamento esperado vs. observado e o node onde travou —
isso acelera muito o diagnóstico.
