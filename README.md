# resumoGrupo — resumo diário de grupo de WhatsApp via IA

Bot que lê as mensagens do seu grupo de WhatsApp ao longo do dia e posta,
num horário definido, um resumo automático: principais tópicos discutidos
(gerado por IA), links compartilhados, e os participantes mais ativos.

Feito com [n8n](https://n8n.io) + um provedor de API WhatsApp não-oficial
(Evolution API por padrão, adaptável pra WAHA ou outros) + Anthropic ou
OpenAI pro resumo de tópicos.

## Pré-requisitos

- Uma instância n8n (**versão 2.7.0 ou superior** — a Data table nativa,
  usada pra guardar as mensagens, só existe a partir dessa versão).
- Um provedor de API WhatsApp não-oficial já conectado com o número que vai
  ficar no grupo (Evolution API é o testado nativamente; veja a seção
  "Adaptar pra outro provedor" se usar WAHA ou outro).
- Uma chave de API da Anthropic ou da OpenAI.
- Ser **admin do grupo** de WhatsApp de destino, se o grupo tiver a opção
  "só admins podem enviar mensagens" ativada.

## Instalação

1. **Crie a Data table.** No seu n8n, vá em Data Tables → New Table, nome
   `resumo_grupo_mensagens`, com as colunas:
   - `remetente_nome` (string)
   - `remetente_jid` (string)
   - `texto` (string)
   - `links` (string)
   - `timestamp` (date)

   Anote o ID da tabela (aparece na URL ou nas configurações da tabela).

2. **Importe os dois workflows.** No n8n, Workflows → Import from File,
   importe `n8n-workflows/captura-workflow.json` e depois
   `n8n-workflows/resumo-diario-workflow.json`.

3. **Configure o workflow de Captura:**
   - Abra o node "Filtrar e extrair" e troque `COLOQUE_O_JID_DO_SEU_GRUPO_AQUI@g.us`
     pelo JID real do seu grupo (veja "Como descobrir o JID do grupo" abaixo).
   - Abra o node "Gravar mensagem" e troque `dataTableId.value` pelo ID da
     tabela que você criou no passo 1.
   - Crie uma credencial do tipo **Header Auth** no n8n, com um nome de
     header e um valor secreto à sua escolha (ex: header `X-Webhook-Secret`,
     valor um token aleatório longo), e vincule essa credencial ao node
     "Webhook Evolution API" (o node já vem configurado com
     `authentication: headerAuth`, só falta apontar pra credencial). Depois,
     configure esse mesmo header no `POST /webhook/set` do seu provedor
     WhatsApp. A Evolution API aceita um campo `headers` nesse endpoint,
     que é onde esse mesmo header entra, pra que só requisições com o
     header certo sejam aceitas pelo seu n8n.
   - Ative o workflow.
   - Copie a URL de produção do webhook (aparece no node "Webhook Evolution
     API" depois de ativado) e configure no seu provedor WhatsApp pra
     mandar eventos de mensagem recebida pra essa URL.

4. **Configure as credenciais no n8n:**
   - Crie uma credencial do tipo Anthropic API (ou HTTP Header Auth, se for
     usar OpenAI — veja "Adaptar pra outro provedor de IA") com sua chave.
   - Crie uma credencial HTTP Header Auth com o header/valor de autenticação
     do seu provedor WhatsApp (pode ser a mesma do webhook, ou outra, a
     depender de como seu provedor autentica chamadas de envio).

5. **Configure o workflow de Resumo Diário:**
   - Node "Ler mensagens de hoje": troque `dataTableId.value` pelo mesmo ID
     do passo 1.
   - Node "Anthropic - resumir tópicos": vincule a credencial criada no
     passo 4, e edite o prompt (dentro do `jsonBody`) trocando
     `'SEU GRUPO AQUI'` pelo nome/contexto real do seu grupo.
   - Node "Enviar mensagem WhatsApp": troque `COLOQUE_A_URL_DE_ENVIO_DO_SEU_PROVEDOR_AQUI`
     e `COLOQUE_O_JID_DO_SEU_GRUPO_AQUI@g.us` pelos valores reais do seu
     provedor, e vincule a credencial.
   - Nos nodes "Formatar - com atividade" e "Formatar - sem atividade",
     troque `'SEU GRUPO AQUI'` pelo nome do seu grupo também.
   - Ajuste o horário no node "Todo dia às 22h" (Schedule Trigger) pro
     horário que você quiser.
   - Ative o workflow.

6. **Teste antes de confiar 100%:** mande uma mensagem de teste no grupo e
   confirme que uma linha apareceu na Data table. Depois, rode o workflow
   de Resumo Diário manualmente (Execute Workflow) com o node de envio
   desativado primeiro, confira o texto gerado, e só depois reative o envio
   real.

## Como descobrir o JID do grupo

Cada provedor tem um jeito diferente. Na Evolution API:
```bash
curl -X GET "https://SUA_INSTANCIA/group/fetchAllGroups/SUA_INSTANCIA?getParticipants=false" \
  -H "apikey: SUA_API_KEY"
```
O campo `id` de cada grupo na resposta é o JID (formato
`NUMERO-TIMESTAMP@g.us`).

## Adaptar pra outro provedor de WhatsApp (WAHA)

[WAHA (WhatsApp HTTP API)](https://waha.devlike.pro/) é uma alternativa
open-source à Evolution API. O formato de payload é diferente, então os
dois pontos do bot que fazem parsing/montagem de mensagem (o node "Filtrar
e extrair" na Captura, e o node "Enviar mensagem WhatsApp" no Resumo
Diário) precisam ser ajustados. Achados da pesquisa na documentação
oficial:

**Webhook de mensagem recebida (evento `message`).** A WAHA envia um POST
com este formato:
```json
{
  "event": "message",
  "session": "default",
  "payload": {
    "id": "true_11111111111@c.us_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA",
    "timestamp": 1667561485,
    "from": "11111111111@c.us",
    "fromMe": true,
    "to": "11111111111@c.us",
    "body": "Hi there!",
    "hasMedia": false,
    "ack": 1
  }
}
```
Em mensagens de grupo, `from`/`to` trazem o JID do grupo (formato
`NUMERO@g.us`, igual à Evolution API), e a WAHA usa um campo `participant`
pra identificar o autor individual dentro do grupo (confirmado na
documentação oficial pra eventos de confirmação de leitura; o padrão é o
mesmo usado pelo protocolo WhatsApp em outros eventos de mensagem.
**Teste e confirme no seu ambiente antes de confiar**, já que a doc pública não
traz um exemplo completo de payload de mensagem de grupo com esse campo
preenchido). O nome de exibição do remetente (equivalente ao `pushName` da
Evolution API) não tem um campo documentado de forma clara na doc pública;
o mais seguro é capturar o payload bruto de uma mensagem real do seu grupo
(logue `$json` no node "Filtrar e extrair" antes de fazer o parsing) e
conferir os campos disponíveis (prováveis candidatos: dentro de
`payload._data`, sub-objeto interno do engine).

Ajuste no node "Filtrar e extrair" (troque o corpo do Code node):
```javascript
const GROUP_JID = 'COLOQUE_O_JID_DO_SEU_GRUPO_AQUI@g.us';

const item = $input.first().json;
const payload = item.body?.payload || item.payload || item;

const remoteJid = payload.from;
const fromMe = payload.fromMe;

if (remoteJid !== GROUP_JID || fromMe === true) {
  return [];
}

const texto = payload.body || '';
if (!texto) {
  return [];
}

const urlRegex = /(https?:\/\/[^\s]+)/g;
const links = (texto.match(urlRegex) || []).join(', ');

const remetenteJid = payload.participant || payload.from || '';
const remetenteNome = remetenteJid; // WAHA não documenta um campo claro de
                                     // nome de exibição; ver nota acima

const timestampIso = new Date(payload.timestamp * 1000).toISOString();

return [{
  json: { remetente_nome: remetenteNome, remetente_jid: remetenteJid, texto, links, timestamp: timestampIso },
}];
```

**Endpoint de envio de mensagem de texto.**
```
POST {SUA_URL_BASE_DA_WAHA}/api/sendText
Header: X-Api-Key: SUA_CHAVE
```
Body:
```json
{
  "session": "default",
  "chatId": "COLOQUE_O_JID_DO_SEU_GRUPO_AQUI@g.us",
  "text": "texto da mensagem"
}
```
Ajuste no node "Enviar mensagem WhatsApp": troque a `url` pro endpoint
acima, o `jsonBody` pra usar `session` e `chatId` em vez de `number`, e a
credencial pra HTTP Header Auth com o header `X-Api-Key`:
```javascript
={{ JSON.stringify({ session: "default", chatId: "COLOQUE_O_JID_DO_SEU_GRUPO_AQUI@g.us", text: $json.mensagemFinal }) }}
```

Fontes consultadas (agosto/2026): [WAHA docs: Receive messages](https://waha.devlike.pro/docs/how-to/receive-messages/),
[WAHA docs: Send messages](https://waha.devlike.pro/docs/how-to/send-messages/),
[WAHA docs: Events](https://waha.devlike.pro/docs/how-to/events/),
[waha-docs, events/index.md no GitHub](https://github.com/devlikeapro/waha-docs/blob/main/content/docs/how-to/events/index.md).

## Adaptar pra outro provedor de IA (OpenAI)

O node "Anthropic - resumir tópicos" chama `POST https://api.anthropic.com/v1/messages`.
Pra usar OpenAI em vez disso:
1. Troque a `url` do node HTTP Request pra `https://api.openai.com/v1/chat/completions`.
2. Troque `nodeCredentialType` pra `openAiApi` (ou HTTP Header Auth com
   `Authorization: Bearer SUA_CHAVE`).
3. Troque o `jsonBody` pro formato da OpenAI:
   ```javascript
   ={{ JSON.stringify({
     model: "gpt-4o-mini",
     messages: [{ role: "user", content: "SEU PROMPT + " + $json.textoConcatenado }]
   }) }}
   ```
4. No node "Formatar - com atividade", troque
   `resp.content?.[0]?.text` por `resp.choices?.[0]?.message?.content`
   (a OpenAI retorna a resposta em `choices[0].message.content`, não em
   `content[0].text` como a Anthropic).

## Filtro de mensagens próprias

Por padrão, o workflow de Captura ignora qualquer mensagem enviada pelo
número conectado à instância (`fromMe === true`). Isso existe pra não
capturar a própria mensagem de resumo diário que o bot posta. Se você
participa ativamente do grupo com esse mesmo número e quer que suas
mensagens entrem no resumo, ajuste a condição no Code node pra filtrar só
mensagens cujo texto comece com o cabeçalho do seu resumo (ex:
`texto.startsWith('📋 Resumo do dia')`) em vez de filtrar `fromMe` inteiro.

## Segurança

O node "Webhook Evolution API" recebe eventos de **toda** a instância
WhatsApp conectada ao seu provedor, não só do grupo configurado. O filtro
por JID acontece depois, dentro do n8n. Por isso:
- O workflow de Captura vem configurado com
  `saveDataSuccessExecution: "none"` e `saveExecutionProgress: false`, pra
  não gravar o conteúdo de conversas privadas (inclusive de fora do grupo)
  no histórico de execuções do n8n. Não reative essas opções sem entender
  essa implicação.
- O webhook vem configurado pra exigir autenticação por header
  (`authentication: headerAuth`). Configure a credencial e o header
  correspondente no seu provedor antes de expor a URL publicamente (veja
  passo 3 da Instalação). Sem isso, qualquer pessoa que descobrir a URL
  pode mandar payloads falsos pro seu webhook.

## Limitações conhecidas desta v1

- Não captura mídia (áudio, imagem, vídeo) — só texto.
- Um grupo por instalação (sem suporte multi-grupo nesta versão).
- Sem alerta automático se a chamada de IA falhar — teste o comportamento
  do dia "sem atividade" antes de confiar no sistema (ver seção de teste
  acima).

## Licença

MIT — veja `LICENSE`.
