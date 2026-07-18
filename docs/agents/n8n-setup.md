# Football World Store — Guia de Implementação N8N

Uso interno | Arquitetura Multi-Agente WhatsApp | Versão 2.0

> **Existe um workflow pronto para importar:** [`../../n8n/footwordstore-workflow.json`](../../n8n/footwordstore-workflow.json).
> Este documento descreve exatamente o que está dentro desse arquivo — não é mais só uma
> especificação em prosa, é a documentação do workflow real. Ver seção "Importação rápida" abaixo
> para colocar no ar.

Resumo rápido do fluxo real:

1. Webhook (WAHA) — recebe o evento de mensagem
2. Set (`Dados`) — normaliza `session`, `chatId`, `pushName`, `message`, `event`
3. Switch — só segue adiante se `event == message`
4. Redis GET — busca se já existe uma rota (`route:{chatId}`) em cache pra essa conversa
5. Se **não** houver rota em cache → AI Agent `fws-central` decide (Gemini + Redis Chat Memory) →
   Code (`Extrair rota`) lê a linha `ROUTE: PALAVRA` da resposta
6. Se **já** houver rota em cache → pula o central e usa a rota salva direto (`Edit Fields`)
7. Switch final (`Roteador`) — 6 saídas: VENDAS / RESERVAS / SUPORTE / FINANCEIRO / HUMANO / CONTINUAR
8. Cada agente especializado é um AI Agent próprio (Gemini + Redis Chat Memory própria) — `fws-vendas`
   e `fws-reservas` têm ferramentas HTTP conectadas (function calling) pra chamar a API pública do
   backend; `fws-suporte` e `fws-financeiro` não têm ferramenta nenhuma (não há endpoint pra elas)
9. WAHA "Send Seen" + "Send Text" — envia a resposta de volta pro WhatsApp

Pontos-chave:

- O `INDEX.md` não vai para o N8N — é referência sua para depuração e mapeamento
- Cada `.md` de agente (com frontmatter incluso) vira o `systemMessage` do nó AI Agent correspondente
- Memória de conversa é a **Redis Chat Memory** nativa do n8n (uma por agente, `sessionKey = chatId`,
  janela de 10 mensagens) — não é um array manual salvo em JSON
- A regra das 3 trocas (ver `fws-*.md`, seção correspondente) **só existe no prompt hoje** — não há
  um contador rígido no workflow que force `HUMANO` depois de N mensagens. Se quiser essa garantia
  fora do LLM, é uma melhoria a adicionar (nó Code contando `exchangeCount` na sessão), não algo já
  implementado no `.json`
- `fws-vendas` e `fws-reservas` precisam de **dado real de estoque** — nunca respondem disponibilidade
  de cabeça, sempre via a ferramenta "Consultar Produtos" (contrato completo em
  [api-endpoints.md](api-endpoints.md))

---

## Importação rápida

1. No n8n, **Workflows → Import from File** → selecione `n8n/footwordstore-workflow.json`.
2. O workflow importa **inativo** (`active: false`) e sem credenciais vinculadas — isso é esperado,
   credenciais nunca são exportadas/importadas entre instâncias n8n. Você precisa:
   - **Redis account** — em todo nó `Redis`, `Redis1` e nos 5 nós `Redis Chat Memory - *` (aponta
     pro Redis do `docker-compose.yml` deste projeto)
   - **WAHA account** — em todo nó `Send Seen*` / `Send a text message*`
   - **Google Gemini(PaLM) Api account** — nos 5 nós `Google Gemini Chat Model*`
3. Configurar as variáveis de ambiente do n8n (usadas pelos nós de ferramenta — ver seção
   "Variáveis de Ambiente" abaixo): `STORE_API_BASE_URL`, `STORE_API_KEY`.
4. Testar manualmente: enviar uma mensagem de teste no WhatsApp conectado ao WAHA e acompanhar a
   execução no canvas do n8n (Executions).
5. Só então **ativar** o workflow.

**Sobre os nós de ferramenta (`Consultar Produtos`, `Consultar Produtos (Reservas)`, `Criar
Reserva`):** foram montados como `HTTP Request Tool` usando `$fromAI(...)` para os parâmetros que o
LLM preenche — esse é o mecanismo oficial do n8n pra tool-calling em nós comuns conectados a um AI
Agent. Os nomes exatos de alguns campos internos podem variar ligeiramente entre versões do n8n; se
algum campo não carregar visualmente após o import, é só reconferir Method/URL/Headers/Body na UI —
2 minutos de ajuste, não invalida o resto do workflow.

---

## Visão Geral da Arquitetura

```text
WhatsApp (cliente)
        │
        ▼
[Webhook] → [Dados] → [Switch: event=message] → [Redis GET route]
                                                        │
                                              ┌─────────┴─────────┐
                                     rota em cache          sem rota em cache
                                              │                    │
                                      [Edit Fields]        [AI Agent: fws-central]
                                              │                    │
                                              │            [Code: Extrair rota]
                                              │                    │
                                              │            rota != CONTINUAR? → [Redis SET route]
                                              │                    │                    │
                                              └────────────────────┴────────────────────┘
                                                                   │
                                                            [Switch: Roteador]
                     ┌───────────┬────────────┬────────────┬────────────┬───────────┐
                     ▼           ▼            ▼            ▼            ▼           ▼
              [Continuar]   [fws-vendas]  [fws-reservas] [fws-suporte] [fws-financeiro] [Humano]
                     │           │            │            │            │           │
                     └───────────┴────────────┴────────────┴────────────┴───────────┘
                                                     │
                                     [WAHA: Send Seen → Send Text]
```

Cada caixa é um nó (ou grupo de nós) real no `footwordstore-workflow.json`.

---

## Estrutura de Nós — Passo a Passo

### Nó 1 — Webhook

**Tipo:** `n8n-nodes-base.webhook`

Recebe o evento do WAHA.

- Method: POST
- Path: `fws/webhook`
- Payload esperado (mesmo formato do WAHA usado no `docker-compose.yml`):

```json
{
  "session": "default",
  "event": "message",
  "payload": {
    "from": "5511999999999@c.us",
    "body": "tem camisa do flamengo tamanho M?",
    "id": "wamid...",
    "fromMe": false,
    "_data": { "Info": { "PushName": "João" } }
  }
}
```

### Nó 2 — Dados (Set)

Normaliza os campos usados no resto do workflow:

| Campo | Expressão |
| --- | --- |
| `session` | `{{ $json.body.session }}` |
| `chatId` | `{{ $json.body.payload.from }}` |
| `pushName` | `{{ $json.body.payload._data.Info.PushName }}` |
| `payload_id` | `{{ $json.body.payload.id }}` |
| `event` | `{{ $json.body.event }}` |
| `message` | `{{ $json.body.payload.body }}` |
| `fromMe` | `{{ $json.body.payload.fromMe }}` |

### Nó 3 — Switch (filtro de evento)

Só deixa passar `event == "message"`. Eventos de outro tipo (ex: confirmação de entrega, presença)
não chegam nos agentes.

### Nó 4 — Redis GET (`route:{chatId}`)

Busca se já existe uma rota ativa em cache pra essa conversa (evita rodar o `fws-central` a cada
mensagem quando o cliente já está no meio de um atendimento).

### Nó 5 — If (rota em cache?)

- **Verdadeiro** (tem rota salva) → `Edit Fields` (seta `route` = valor do Redis) → direto pro
  `Roteador`, sem rodar o central de novo.
- **Falso** (sem rota salva) → roda o `AI Agent: fws-central`.

### Nó 6 — AI Agent: fws-central

**Tipo:** `@n8n/n8n-nodes-langchain.agent`, conectado a:
- `ai_languageModel` → Google Gemini Chat Model (`temperature: 0`, decisão determinística)
- `ai_memory` → Redis Chat Memory (`sessionKey = chatId`, janela de 10 mensagens)

`systemMessage` = conteúdo integral de [fws-central.md](fws-central.md), incluindo o frontmatter.

### Nó 7 — Code: Extrair rota

Lê a linha `ROUTE: PALAVRA` da resposta do central via regex, separa `route` (a palavra) de
`agentResponse` (o texto sem a linha de roteamento):

```javascript
const output = $input.item.json.output || '';
const match = output.match(/ROUTE:\s*(VENDAS|RESERVAS|SUPORTE|FINANCEIRO|HUMANO|CONTINUAR)/i);
const route = match ? match[1].toUpperCase().trim() : 'CONTINUAR';
const agentResponse = output
  .replace(/\nROUTE:\s*(VENDAS|RESERVAS|SUPORTE|FINANCEIRO|HUMANO|CONTINUAR).*/im, '')
  .replace(/\n(VENDAS|RESERVAS|SUPORTE|FINANCEIRO|HUMANO|CONTINUAR)\s*$/im, '')
  .trim();
return [{ json: { ...$input.item.json, route, agentResponse } }];
```

### Nó 8 — If1 (salvar rota?)

- Se `route != CONTINUAR` → `Redis SET route:{chatId}` (TTL 3600s) antes de seguir pro `Roteador` —
  fixa o agente atual pra próxima mensagem não precisar passar pelo central de novo.
- Se `route == CONTINUAR` → segue direto pro `Roteador` sem salvar nada.

### Nó 9 — Roteador (Switch final, 6 saídas)

Direciona conforme `route`:

| Saída | Rota | Destino |
| --- | --- | --- |
| Continuar | `CONTINUAR` | `Send Seen` → `Send a text message` (envia a resposta que o **próprio central** já gerou) |
| Vendas | `VENDAS` | AI Agent `fws-vendas` |
| Reservas | `RESERVAS` | AI Agent `fws-reservas` |
| Suporte | `SUPORTE` | AI Agent `fws-suporte` |
| Financeiro | `FINANCEIRO` | AI Agent `fws-financeiro` |
| Humano | `HUMANO` | `Send a text message - Humano` — texto fixo, **não invoca nenhum LLM** |

### Nós 10 — Agentes especializados

Cada um (`fws-vendas`, `fws-reservas`, `fws-suporte`, `fws-financeiro`) é um AI Agent com:

- `systemMessage` = conteúdo integral do `.md` correspondente
- `ai_languageModel` próprio → Google Gemini Chat Model (`temperature: 0.3` em Vendas, `0.2` nos
  demais — Vendas precisa de um pouco mais de naturalidade na conversa consultiva)
- `ai_memory` própria → Redis Chat Memory (mesma `sessionKey = chatId`, janela de 10 mensagens —
  memória por agente, não compartilhada com o central)
- Texto de entrada inclui um cabeçalho de contexto:

```
CONTEXTO DO ATENDIMENTO:
- Intenção identificada pelo agente central: VENDAS
- Mensagem do usuário: {{ mensagem }}
```

**Ferramentas conectadas (`ai_tool`):**

| Agente | Ferramentas |
| --- | --- |
| `fws-vendas` | `Consultar Produtos` (GET `/api/v1/public/products`) |
| `fws-reservas` | `Consultar Produtos (Reservas)` + `Criar Reserva` (POST `/api/v1/public/reservations`) |
| `fws-suporte` | nenhuma |
| `fws-financeiro` | nenhuma |

Contrato completo de cada endpoint em [api-endpoints.md](api-endpoints.md).

### Nó 11 — WAHA: Send Seen → Send Text

Cada branch (Continuar, Vendas, Reservas, Suporte, Financeiro) termina em `Send Seen` (marca a
mensagem como lida) seguido de `Send a text message` (envia `agentResponse`/`output` do agente pro
WhatsApp via WAHA). O branch `Humano` pula direto pro `Send a text message - Humano` com um texto
fixo, sem gerar nada via LLM.

### Nó 12 — WAHA: Foto do produto (só no branch Vendas)

Depois de `Send a text message - fws-vendas`, tem mais dois nós:

1. **`Tem 1 Produto com Foto? - fws-vendas`** (`n8n-nodes-base.if`) — lê o resultado da última
   chamada da tool `Consultar Produtos` nessa execução (`$('Consultar Produtos').item.json.data`) e
   só segue pro `true` quando veio **exatamente 1 produto** com `photoUrl` preenchido. A expressão
   está envolta em `try/catch` porque, se o LLM não chamou `Consultar Produtos` nessa execução (ex:
   mensagem de saudação), referenciar o node por nome quebraria a expressão — nesse caso o `catch`
   devolve `false` e o fluxo simplesmente não manda foto.
2. **`Enviar Foto do Produto - fws-vendas`** (`n8n-nodes-waha.WAHA`, resource `Chatting`, operation
   `Send Image`) — manda o `photoUrl` do produto (vem do bucket S3 real, ver
   [api-endpoints.md](api-endpoints.md)) como `file.url`, com legenda contendo nome, clube, tamanho
   e preço.

**Limitação intencional (v1):** só manda foto quando a busca retornou um produto só, exatamente pra
evitar rajada de imagem numa lista grande — se o cliente pediu "camisas do Flamengo" e vieram 5
resultados, só a mensagem de texto é enviada, sem foto. Se quiser cobrir também o caso de poucos
resultados (2-3) mandando várias fotos, precisa de um node de loop (`Split In Batches`) — não
implementado ainda.

**Não testado ponta a ponta com WhatsApp real** — a operação `Send Image` e os nomes exatos dos
campos (`session`, `chatId`, `file`, `caption`) foram conferidos direto no código-fonte do pacote
`n8n-nodes-waha` (schema gerado a partir do `openapi.json` do WAHA), não por execução real do
workflow. Testar mandando uma mensagem que bata em exatamente 1 produto com foto cadastrada (ver
produto de teste `FWS-0002` no catálogo) antes de confiar no fluxo em produção.

---

## Ferramentas HTTP (Tool nodes)

**Tipo:** `n8n-nodes-base.httpRequestTool`, conectadas via `ai_tool` ao respectivo AI Agent.

### Consultar Produtos (fws-vendas e fws-reservas)

- Method: GET
- URL: `{{ $env.STORE_API_BASE_URL }}/api/v1/public/products`
- Query params preenchidos pelo LLM via `$fromAI(...)`: `search`, `category`, `size`,
  `clubOrBrand`, `stockStatus`
- Header: `x-api-key: {{ $env.STORE_API_KEY }}`

### Criar Reserva (fws-reservas)

- Method: POST
- URL: `{{ $env.STORE_API_BASE_URL }}/api/v1/public/reservations`
- Body JSON preenchido pelo LLM via `$fromAI(...)`: `productId`, `quantity`, `customerName`,
  `customerWhatsapp`, `customerEmail`, `notes`
- Headers: `x-api-key` + `Content-Type: application/json`

`$fromAI(key, description, type, defaultValue)` é a função nativa do n8n pra deixar o LLM preencher
um parâmetro — funciona em qualquer campo de um nó conectado como tool a um AI Agent.

### Tratamento de falha técnica (`onError`)

Os três nós de ferramenta (`Consultar Produtos`, `Consultar Produtos (Reservas)`, `Criar Reserva`)
têm `"onError": "continueRegularOutput"`. Sem isso, uma falha de rede/timeout/erro do backend
derrubaria a execução inteira do workflow e o cliente não receberia **nenhuma** resposta no
WhatsApp. Com `continueRegularOutput`, a falha vira um item de saída com campo `error` — o LLM
recebe isso como resultado da ferramenta e reage conforme a instrução dos respectivos `.md`
(seção "REGRA CRÍTICA"): responde a mensagem fixa de fallback e escala pra humano, em vez de
travar ou inventar uma resposta.

> Isso é diferente de "nenhum produto encontrado" (busca válida, sem resultado) — esse caso não
> tem `error`, é tratado separadamente em cada `.md` (resposta mais suave, sem escalar).

---

## Variáveis de Ambiente (.env do N8N)

Usadas dentro das expressões dos nós de ferramenta:

| Variável | Uso |
| --- | --- |
| `STORE_API_BASE_URL` | URL base do backend (módulo `Public` — produtos e reservas) |
| `STORE_API_KEY` | API key (`x-api-key`) pra autenticar no módulo `Public` do backend |

Redis, WAHA e Gemini **não** são variáveis de ambiente — são **credentials** do n8n, configuradas
direto na UI (Credentials → New) e vinculadas em cada nó após o import (ver "Importação rápida").

---

## Regra das 3 Trocas — hoje é só prompt

A instrução "depois de 3 trocas sem resolução, oferecer humano" está escrita nos `.md` de cada
agente, mas **o workflow não tem um contador que force isso** — depende do LLM seguir a instrução.
Se quiser essa garantia fora do modelo (mais robusta, não depende do LLM "lembrar"), dá pra adicionar
um nó Code antes do `Roteador` que lê um `exchangeCount` da sessão Redis e força `route = HUMANO`
acima de um limite — isso não está implementado no `.json` atual, é uma melhoria em aberto.

---

## Estrutura de Arquivos do Projeto

```text
docs/agents/
├── INDEX.md              ← índice invertido mestre (para consulta interna)
├── fws-central.md         ← system prompt do agente roteador
├── fws-vendas.md          ← system prompt do agente de catálogo/produtos
├── fws-reservas.md        ← system prompt do agente de reserva
├── fws-suporte.md         ← system prompt do agente de pós-venda/cadastro
├── fws-financeiro.md      ← system prompt do agente de pagamento
├── api-endpoints.md       ← contrato dos endpoints do backend consumidos pelo n8n
└── n8n-setup.md           ← este arquivo

n8n/
└── footwordstore-workflow.json   ← workflow importável (nodes, conexões, tool schemas)

prisma/
└── schema.prisma          ← fonte de verdade do sistema de estoque/vendas real (uso interno do backend)
```

O [INDEX.md](INDEX.md) **não é um system prompt** — é uma referência interna para você montar ou
depurar os roteamentos. Não cole ele no N8N (ele já não está no `.json`).

---

## Dicas de Depuração

**O roteador está mandando para o agente errado?**
Abra a execução no n8n e olhe o output do nó `Extrair rota` — confirme que `route` saiu com o valor
esperado. Verifique se o Gemini Chat Model do `fws-central` está com `temperature: 0`.

**O agente de Vendas está inventando estoque?**
Verifique se a ferramenta `Consultar Produtos` está de fato conectada (`ai_tool`) ao nó `fws-vendas`
no canvas — sem essa conexão, o LLM não tem como chamar a API e vai alucinar disponibilidade mesmo
seguindo a instrução do `.md`.

**A ferramenta HTTP não está sendo chamada pelo LLM?**
Confira o `toolDescription` — é ele que o modelo usa pra decidir *quando* chamar a ferramenta. Se
estiver vago demais, o LLM pode responder sem consultar.

**Erro de autenticação na chamada da API pública?**
Confirme `STORE_API_KEY` nas variáveis de ambiente do n8n e que o backend tem esse mesmo valor
configurado como `PUBLIC_API_KEY` (ver [api-endpoints.md](api-endpoints.md)).

**O bot travou ou o cliente não recebeu resposta nenhuma quando o backend caiu?**
Confira se os 3 nós de ferramenta (`Consultar Produtos`, `Consultar Produtos (Reservas)`,
`Criar Reserva`) ainda têm `onError: continueRegularOutput` — sem isso a execução para no meio e
nada é enviado pro WhatsApp. Se o `onError` estiver certo mas o cliente recebeu uma resposta
estranha em vez da mensagem fixa de fallback, o LLM não seguiu a instrução — reforce a seção
"REGRA CRÍTICA" do `.md` correspondente.

**O agente está ignorando as instruções do `.md`?**
Reduza a janela de contexto da Redis Chat Memory (`contextWindowLength`) — histórico muito longo
consome tokens e dilui as instruções do `systemMessage`.

**Respostas genéricas ou fora do escopo?**
Verifique se o `systemMessage` inclui a seção "O que você NÃO faz" do respectivo `.md`. É ela que
impede respostas fora do escopo.

**Loop de roteamento (CONTINUAR toda hora, nunca sai do central)?**
Verifique se o `Redis SET route:{chatId}` (nó `Redis1`) está de fato gravando — sem isso, toda
mensagem nova cai em "sem rota em cache" e roda o central de novo.

---

*Football World Store — N8N Setup Guide — Documento de uso interno.*
*Atualizar sempre que o `footwordstore-workflow.json` for modificado.*
