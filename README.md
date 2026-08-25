# resumoGrupo — resumo diário e semanal de grupo de WhatsApp via IA

Bot que lê as mensagens do seu grupo de WhatsApp ao longo do dia e posta,
num horário definido, um resumo automático com os principais tópicos
discutidos (gerado por IA, um parágrafo por assunto), os links
compartilhados e os participantes mais ativos. Inclui também um resumo
semanal opcional.

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
   - Abra o node "Webhook Evolution API" e troque o path
     `TROQUE-POR-UM-PATH-ALEATORIO-SEU` por um valor aleatório e longo, só
     seu (ex: `captura-` seguido de um UUID). **Esse path é a autenticação
     do webhook**: quem não souber a URL não consegue mandar payload falso
     pro seu n8n. Não use um path curto nem adivinhável, e trate essa URL
     como senha.
     Não configure Header Auth aqui. A Evolution API self-hosted **não
     envia headers customizados** no webhook (o campo `headers` existe no
     schema e aceita ser gravado, mas o recurso é da versão Cloud). Se você
     exigir um header que nunca chega, o n8n passa a responder 401 e a
     captura para em silêncio, sem erro visível em lugar nenhum.
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

## Mensagens enviadas pelo próprio número

O workflow de Captura **grava** as mensagens enviadas pelo número conectado
à instância, e descarta só o que o próprio bot publica, reconhecido pelo
cabeçalho do resumo (`📋 Resumo do dia` e `📅 Resumo da semana`). Sem esse
descarte, o resumo de ontem viraria assunto do resumo de hoje.

Isso serve aos dois cenários. Se a instância é um número dedicado ao bot, as
únicas mensagens `fromMe` que existem são os próprios resumos, então o
resultado é o mesmo de ignorar tudo. Se a instância é o seu número pessoal e
você participa do grupo, suas mensagens entram nos tópicos e no Top como as
de qualquer participante.

Dois detalhes da implementação, no node "Filtrar e extrair":

- A comparação remove os asteriscos antes de comparar, então continua
  funcionando se você mudar o negrito do cabeçalho.
- O descarte só vale com `fromMe === true`. O mesmo cabeçalho colado por
  outra pessoa é mensagem normal e entra no resumo.

Se você mudar o texto do cabeçalho no node de formatação, **mude também a
constante `PREFIXOS_DO_BOT`** no node "Filtrar e extrair", senão o bot passa
a resumir os próprios resumos.

O nome usado nas mensagens próprias é a constante `NOME_PROPRIO`, e não o
`pushName`, que não chega de forma confiável em `fromMe`. Troque
`SEU NOME AQUI` pelo nome que você quer ver aparecer no Top.

## Segurança

O node "Webhook Evolution API" recebe eventos de **toda** a instância
WhatsApp conectada ao seu provedor, não só do grupo configurado. O filtro
por JID acontece depois, dentro do n8n. Por isso:
- O workflow de Captura vem configurado com
  `saveDataSuccessExecution: "none"` e `saveExecutionProgress: false`, pra
  não gravar o conteúdo de conversas privadas (inclusive de fora do grupo)
  no histórico de execuções do n8n. Não reative essas opções sem entender
  essa implicação.
- O webhook é protegido por **path secreto**, não por header. Versões
  anteriores deste pacote vinham com `authentication: headerAuth` e o README
  mandava configurar o header no `POST /webhook/set` da Evolution. Isso não
  funciona: a Evolution self-hosted não envia headers customizados, então o
  n8n respondia 401 a todas as chamadas e a captura ficava muda. Se você
  instalou uma versão anterior e o bot nunca gravou nada, é isso.
  Troque o path por um valor aleatório longo e trate a URL como senha.
- Desligar o histórico de execuções tem um efeito colateral que vale saber:
  no n8n, `saveDataSuccessExecution: "none"` faz a execução sumir **inteira**
  da lista, não só o conteúdo dela. Depois disso, a única forma de saber se a
  captura está viva é olhar as linhas gravadas na Data table.

## Confiabilidade

Dois ajustes garantem que o resumo diário sempre seja postado, mesmo em
dias sem atividade ou com falha externa:
- O node "Ler mensagens de hoje" vem com `alwaysOutputData: true`. Sem
  isso, quando não há nenhuma mensagem no dia, o n8n interrompe a execução
  ali mesmo e a mensagem de "sem atividade" nunca é postada.
- O node "Calcular estatísticas" descarta o item vazio que essa flag produz.
  As duas coisas andam juntas e **não podem ser separadas**: com
  `alwaysOutputData` ligado e a checagem antiga de lista vazia, um dia sem
  mensagem publicava no grupo um participante fantasma,
  `1. Desconhecido (1 msg)`, em vez do aviso de dia parado. Versões
  anteriores deste pacote tinham exatamente essa combinação.
- O node "Filtrar e extrair" tolera `messageTimestamp` ausente ou malformado.
  Antes, `new Date(NaN).toISOString()` lançava e a captura perdia a mensagem.
- O node "Anthropic - resumir tópicos" vem com
  `onError: "continueRegularOutput"`. Se a chamada de IA falhar (chave sem
  crédito, API fora do ar, etc.), a execução segue e o node "Formatar - com
  atividade" usa o texto de fallback ("Não foi possível gerar o resumo
  hoje.") em vez de derrubar a execução inteira — você ainda recebe os
  links e o Top 5, calculados sem depender da IA.

## Resumo semanal

O pacote inclui `resumo-semanal-workflow.json`, que roda uma vez por semana
(domingo às 9h por padrão) e resume de segunda a domingo, agrupando por
assunto em vez de por dia. A instalação é a mesma do diário: trocar os
placeholders, apontar as credenciais e ativar. Ele lê a mesma Data table, não
precisa de tabela própria.

Se você não quiser o semanal, simplesmente não importe esse arquivo.

## Testando antes de soltar no grupo

Os Code nodes são JavaScript puro e podem ser testados fora do n8n, o que
evita descobrir um erro de formatação com o grupo inteiro assistindo.

O jeito mais simples: abra o JSON do workflow, copie o `jsCode` do node que
quer testar e rode com `node`, passando um `$input` falso. Para os nodes de
formatação, que usam `$now`, instale `luxon` e passe um `DateTime` no lugar.

Antes da primeira execução real, vale mandar o resumo para o seu número
privado em vez do grupo: troque o destino no node de envio, dispare o
workflow uma vez e veja como o negrito, o itálico e os separadores realmente
renderizam no WhatsApp. Só então aponte para o grupo.

## Limitações conhecidas desta v1

- Não captura mídia (áudio, imagem, vídeo) — só texto.
- Um grupo por instalação (sem suporte multi-grupo nesta versão).
- Se a chamada de IA falhar, o resumo de tópicos vira um texto de fallback
  (ver seção "Confiabilidade" acima) — não há alerta automático avisando
  que isso aconteceu, então vale conferir de vez em quando se o resumo
  ainda está vindo com o parágrafo de tópicos normal.

## Licença

MIT — veja `LICENSE`.
