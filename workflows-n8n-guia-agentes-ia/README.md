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

## Versão do n8n

Os workflows foram testados no n8n 1.123.82, a mesma versão citada na Nota legal do livro. No n8n 2.x, dois cuidados são necessários antes de testar:

- **Variáveis de ambiente.** A versão 2 bloqueia por padrão o acesso a `$env`. Defina `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` no ambiente da instância (veja o passo 4 do `GUIA_AMBIENTE_ESTUDO.md`).
- **Publicação.** Na versão 2, salvar e publicar são etapas separadas. Depois de importar, use **Publish** para o workflow receber chamadas de produção.

Os nós **HTTP Request Tool** (legado) e as versões 1.x do **AI Agent** usados aqui funcionam na 2.x, mas foram removidos no n8n 3.0. Uma versão destes arquivos para a 3.x será publicada neste repositório.

## Diferenças em relação aos fluxos do livro

Os fluxos em YAML dos capítulos são didáticos. Estes arquivos são implementações de partida e, em alguns detalhes, fazem escolhas diferentes:

- `handoff-humano-redis.json`: TTL do modo humano de 1 hora (no livro, 30 minutos).
- `lembrete-agendamento.json`: verificação a cada 30 minutos (no livro, a cada hora).
- `faq-dinamico-sheets.json`: o FAQ sincronizado fica no Postgres (no livro, em cache no Redis).
- `avaliacao-pipeline.json`: avaliação diária às 6h (no livro, semanal).
- `error-handler-backoff.json`: até 3 novas tentativas por hora, com espera de 30 s, 60 s e 120 s mais um valor aleatório, e alerta ao operador pelo WhatsApp.

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
