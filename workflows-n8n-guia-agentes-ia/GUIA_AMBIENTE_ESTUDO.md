# Guia do Ambiente de Estudo — Guia Enterprise de Boas Práticas para Agentes de IA

Este guia monta um ambiente **isolado e independente**, só pra rodar os
workflows do Apêndice B do livro. Não usa nem toca em nenhum banco de
produção — é um schema próprio, com dados de exemplo, pensado pra você
testar os conceitos do livro sem precisar ter um sistema real por trás.

---

## 1. Suba um banco Postgres dedicado

Se você já roda Postgres localmente (Docker, instalação nativa, ou
serviço gerenciado), crie um database novo só pra isso:

```sql
CREATE DATABASE guia_agentes_estudo;
```

Se ainda não tem Postgres rodando, o caminho mais rápido é Docker:

```bash
docker run -d --name postgres-estudo \
  -e POSTGRES_PASSWORD=estudo123 \
  -e POSTGRES_DB=guia_agentes_estudo \
  -p 5432:5432 \
  postgres:16
```

## 2. Rode o schema-estudo.sql

Conecte no banco `guia_agentes_estudo` e rode o arquivo `schema-estudo.sql`
inteiro. Ele cria 8 tabelas (`business`, `faq`, `execution_logs`,
`conversation_messages`, `conversation_evals`, `appointments`,
`document_extractions`, `leads`) e já insere um negócio de exemplo, FAQ,
um agendamento e uma conversa de teste.

```bash
psql "postgresql://postgres:estudo123@localhost:5432/guia_agentes_estudo" \
  -f schema-estudo.sql
```

Ou, se preferir, abra o arquivo num cliente gráfico (DBeaver, TablePlus,
pgAdmin) e execute o conteúdo inteiro de uma vez.

**Confirme que populou** rodando:
```sql
SELECT * FROM business;
```
Você deve ver uma linha: "Clínica Exemplo (Estudo)".

## 3. Configure a credencial Postgres no n8n

No n8n: **Credentials → New → Postgres**. Aponte pro banco
`guia_agentes_estudo` que você acabou de criar. Essa é a credencial que
você vai selecionar em todo node Postgres dos workflows importados.

## 4. Configure as variáveis de ambiente

No n8n: **Settings → Variables** (ou `.env` da sua instância):

| Variável | Valor de exemplo | Obrigatória para |
|---|---|---|
| `WAHA_BASE_URL` | `https://sua-instancia-waha.com` | Todos os fluxos de WhatsApp |
| `WAHA_API_KEY` | (sua API key do WAHA) | Todos os fluxos de WhatsApp |
| `WAHA_DEFAULT_SESSION` | `default` | 04, 06, 07, 08 |
| `OPERATOR_WHATSAPP` | seu número, formato `5511999999999@c.us` | 04, 06, 07 |
| `ANTHROPIC_API_KEY` | (sua chave da API da Anthropic) | 07, 10 |
| `API_BASE_URL` | URL base do seu n8n + `/webhook` (ex: `https://seu-n8n.com/webhook`) | 01, 02, sub-agente de agendamento |
| `API_INTERNAL_KEY` | qualquer string secreta à sua escolha | 01, 02, sub-agente de agendamento |
| `N8N_BASE_URL` | URL base do seu n8n + `/webhook` (mesmo valor de `API_BASE_URL` no ambiente de estudo) | 04, multi-agente |
| `N8N_API_BASE_URL` / `N8N_API_KEY` | API própria do n8n (Settings → API) | 06 |
| `FAQ_SHEET_ID` | ID de uma planilha Google Sheets sua | 03 (opcional — veja nota abaixo) |
| `DEFAULT_BUSINESS_ID` | `1` (o negócio de exemplo criado no passo 2) | 03, API de estudo |
| `DEFAULT_CALENDAR_ID` | e-mail do Google Calendar de teste | sub-agente de agendamento, 09 |

No ambiente de estudo, **`API_BASE_URL` e `N8N_BASE_URL` apontam pro
próprio n8n** — é o workflow `ambiente-estudo-api.json` (passo 6) que
recebe essas chamadas, não um backend separado.

## 5. Configure a credencial Anthropic no n8n

**Credentials → New → Anthropic**. Cole sua chave da API. É a credencial
usada em todo node `Anthropic Chat Model`.

## 6. Importe os workflows, nesta ordem

A ordem importa porque alguns workflows chamam webhooks de outros.

1. **`ambiente-estudo-api.json`** — importe e **ative**. Sem isso, os workflows 01, 02 e o sub-agente de agendamento não têm o que chamar.
2. **`faq-dinamico-sheets.json`** (03) — importe e ative, mesmo sem configurar o Google Sheets ainda (a consulta via webhook já funciona com os dados de exemplo inseridos direto no banco).
3. **`handoff-humano-redis.json`** (04) — precisa de uma credencial Redis também (veja passo 7).
4. **`freebusy-cache.json`** (09) — precisa de credencial Google Calendar OAuth2 (opcional pra testar só os outros fluxos).
5. Os 4 sub-agentes (`sub-agente-triagem-exemplo.json`, `sub-agente-faq-exemplo.json`, `sub-agente-agendamento-exemplo.json`, `sub-agente-suporte-exemplo.json`) — importe todos, **não precisa ativar** (são chamados como sub-workflow, não têm webhook próprio).
6. **`multi-agent-manager.json`** (05) — depois de importar os 4 sub-agentes acima, copie o ID de cada um (aparece na URL do workflow no n8n) e cole nos campos `workflowId` dos nodes `Tool` correspondentes neste workflow.
7. Os demais (`atendimento-smb-base.json`, `qualificacao-leads.json`, `error-handler-backoff.json`, `avaliacao-pipeline.json`, `lembrete-agendamento.json`, `documento-analise.json`) — importe na ordem que preferir.

## 7. Credencial Redis (só necessária para os workflows 04 e 09)

Se você não tem Redis rodando:
```bash
docker run -d --name redis-estudo -p 6379:6379 redis:7
```
No n8n: **Credentials → New → Redis**, host `localhost`, porta `6379`.

## 8. Teste cada workflow isoladamente

Não tente rodar os 14 de uma vez. Sugestão de ordem de teste:

1. **`ambiente-estudo-api.json`**: rode manualmente uma chamada
   `GET {{API_BASE_URL}}/business-info/1` — deve retornar os dados da
   "Clínica Exemplo".
2. **`faq-dinamico-sheets.json`**: rode `POST {{API_BASE_URL}}/faq-consultar`
   com `{"termo_busca": "horário"}` — deve retornar a FAQ de horário de
   funcionamento.
3. **`avaliacao-pipeline.json`**: execute manualmente (Execute Workflow no
   n8n) — já tem uma conversa de exemplo no banco pra avaliar, não precisa
   de WhatsApp real.
4. **`atendimento-smb-base.json`**: esse já precisa do WAHA configurado
   de verdade com uma sessão ativa — é o primeiro fluxo pra testar
   conversando de fato.

## 9. Isolamento de ambientes

Se em algum momento você quiser usar esses workflows como base pra um
projeto real (não mais estudo), **não aponte pro banco `guia_agentes_estudo`**
— crie um schema novo e adapte o modelo de dados às necessidades reais
do seu projeto. O schema deste guia é intencionalmente simplificado
(tabela `business` achatada, sem separação de configuração/informações
como um sistema em produção normalmente teria) para manter o foco
didático nos conceitos do livro.
