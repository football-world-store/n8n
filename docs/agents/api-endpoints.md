# Football World Store — Endpoints da API (consumidos pelo n8n)

Uso interno | Referência de integração | Versão 1.0

Este documento lista exatamente quais endpoints do backend o workflow n8n do chatbot consome. O
backend expõe um módulo `Public` dedicado a isso — autenticado por API key estática, sem exigir
login de usuário. **Nenhum outro módulo é acessado pelo chatbot** (Sales, Users, Customers,
Dashboard, Alerts, Audit são exclusivos do painel administrativo, OWNER/EMPLOYEE autenticado).

---

## Autenticação

Todas as chamadas do n8n para o backend usam o header:

```
x-api-key: {{ $env.STORE_API_KEY }}
```

| Variável de ambiente (n8n) | Uso |
| --- | --- |
| `STORE_API_BASE_URL` | URL base do backend (ex: `https://api.footballworldstore.com`) |
| `STORE_API_KEY` | Valor configurado como `PUBLIC_API_KEY` no backend |

---

## GET /api/v1/public/products

**Usado por:** `fws-vendas` (busca de catálogo, preço, disponibilidade) e `fws-reservas`
(confirmação de estoque antes de reservar).

**Quando chamar:** sempre que o cliente perguntar por um produto, time, categoria ou tamanho —
nunca responder disponibilidade ou preço sem essa consulta primeiro.

Query params:

| Param | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `search` | string | Não | Busca em nome, código (FWS-XXXX) ou clube/marca |
| `category` | enum | Não | `CAMISA` \| `SHORT` \| `MEIAO` \| `AGASALHO` \| `ACESSORIO` \| `CALCADO` |
| `size` | string | Não | Tamanho (P, M, G, GG, 38, 42...) |
| `clubOrBrand` | string | Não | Clube ou marca |
| `stockStatus` | enum | Não | `AVAILABLE` \| `LOW_STOCK` \| `OUT_OF_STOCK` |

Resposta (200):

```json
{
  "data": [
    {
      "id": "uuid",
      "internalCode": "FWS-0001",
      "name": "Camisa Flamengo I 2024/25",
      "clubOrBrand": "Flamengo",
      "category": "CAMISA",
      "size": "M",
      "photoUrl": null,
      "salePrice": 269.90,
      "quantity": 10,
      "stockStatus": "AVAILABLE"
    }
  ],
  "total": 1,
  "truncated": false
}
```

> `costPrice` nunca é retornado por este endpoint — o bot nunca tem acesso ao custo do produto.

**Limite de resultados:** a busca retorna no máximo 20 produtos por chamada. Quando o resultado é
cortado nesse limite, `truncated` vem `true` — o agente deve pedir pro cliente refinar a busca
(time, categoria, tamanho) em vez de apresentar a lista como se fosse o catálogo inteiro.

**`stockStatus` — como o agente deve reagir:**

| Valor | Significado | Reação do fws-vendas |
| --- | --- | --- |
| `AVAILABLE` | Estoque acima do mínimo | Confirma disponibilidade normalmente |
| `LOW_STOCK` | Tem estoque, mas abaixo do mínimo | Confirma disponibilidade e sinaliza "últimas unidades" |
| `OUT_OF_STOCK` | Sem estoque | Informa que zerou, oferece alternativa — nunca oferece reserva |

---

## POST /api/v1/public/reservations

**Usado por:** `fws-reservas`, depois que o produto foi confirmado disponível via
`GET /public/products`.

Body:

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `productId` | UUID | Sim | Vem do `id` retornado pela busca de produtos |
| `quantity` | integer | Sim | Mínimo 1 |
| `customerName` | string | Sim | Nome do cliente |
| `customerWhatsapp` | string | Sim | Número com DDD e código do país, só dígitos |
| `customerEmail` | string | Não | Opcional |
| `notes` | string | Não | Observação livre |

Resposta (201):

```json
{
  "data": {
    "id": "uuid",
    "productName": "Camisa Flamengo I 2024/25",
    "productCode": "FWS-0001",
    "size": "M",
    "quantity": 1,
    "customerName": "João Silva",
    "status": "PENDING",
    "expiresAt": "2026-04-28T18:22:43.417Z",
    "message": "Reserva criada! Nossa equipe confirmará em breve pelo WhatsApp. Válida por 24h."
  }
}
```

Erros:

| Código | HTTP | Quando | Como o agente reage |
| --- | --- | --- | --- |
| `PRODUCT_NOT_FOUND` | 404 | Produto não existe ou está inativo | Route de volta pra `fws-vendas` |
| `PRODUCT_OUT_OF_STOCK` | 400 | Estoque zerado | Informa que zerou, oferece alternativa |
| `INSUFFICIENT_STOCK` | 400 | Quantidade pedida maior que a disponível | Informa a quantidade máxima disponível |
| `INVALID_API_KEY` | 401 | Falha de configuração do n8n | Não expor ao cliente — alertar equipe técnica |

**Importante:** a reserva **não desconta o estoque** — é um pedido de hold que a loja confirma
manualmente pelo painel interno. Prazo de validade é sempre **24h**, fixo pelo backend — o agente
não pode prometer prazo diferente. O status retornado é `PENDING` (não confundir com o valor em
pt-BR `AGUARDANDO` usado no banco — a API sempre responde em inglês).

---

## Endpoint futuro (não implementado) — Lista de espera

> **Status: proposta de contrato, não existe no backend ainda.** Documentado aqui pra já deixar o
> formato alinhado com o padrão do módulo `Public` (mesma auth, mesma filosofia de não expor dado
> sensível) quando/se for implementado. `fws-reservas.md` (seção "Lista de espera") já assume esse
> contrato — até lá, o agente resolve de forma manual (coleta + escala pra `HUMANO`).

Cobre o caso de um produto `OUT_OF_STOCK`: o cliente quer ser avisado quando o estoque for
reposto. É um modelo diferente da reserva — não há produto pra separar agora.

### POST /api/v1/public/waitlist (proposto)

Body:

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `productId` | UUID | Sim | Vem do `id` retornado pela busca de produtos |
| `customerName` | string | Sim | Nome do cliente |
| `customerWhatsapp` | string | Sim | Número com DDD e código do país, só dígitos |
| `customerEmail` | string | Não | Opcional |

Resposta (201) proposta:

```json
{
  "data": {
    "id": "uuid",
    "productName": "Camisa Flamengo I 2024/25",
    "productCode": "FWS-0001",
    "size": "M",
    "customerName": "João Silva",
    "message": "Anotado! Avisamos assim que chegar."
  }
}
```

Erros propostos: `PRODUCT_NOT_FOUND` (404), `INVALID_API_KEY` (401).

**Em aberto — decidir antes de implementar:** como a notificação de "chegou" realmente sai pro
cliente quando o estoque for reposto. Duas opções, sem uma escolhida ainda:
- O backend dispara um webhook pro n8n quando uma `StockEntry` reabastece um produto com fila de
  espera (o módulo `Alerts` já detecta isso via `checkStockAlerts()` — daria pra estender);
- Um workflow n8n separado faz polling periódico em um endpoint tipo
  `GET /api/v1/public/waitlist/ready`.

Isso é uma decisão de arquitetura pro backend, não do escopo deste documento — só sinalizando que o
contrato de criação (`POST`) sozinho não resolve o problema inteiro.

---

## Fora do escopo do chatbot (hoje)

O bot **não** consulta status de pedido, histórico de compras nem atualiza cadastro de cliente via
API — não existe endpoint público para isso hoje. Esses casos (ver
[`fws-suporte.md`](fws-suporte.md)) são sempre coletados em texto e escalados para `HUMANO`.

Se no futuro isso for automatizado, seguir o mesmo padrão do módulo `Public`: endpoint dedicado,
autenticado por API key, sem expor dados sensíveis (sem `costPrice`, sem dados de outros clientes).

---

## Tabela de valores e parâmetros operacionais

Parâmetros de negócio que **não vêm da API** — usados principalmente por `fws-financeiro` e
`fws-reservas`. Preencher conforme a loja define cada valor; os agentes devem usar o que estiver
aqui em vez de inventar condição.

| Parâmetro | Valor | Observação |
| --- | --- | --- |
| Prazo de validade da reserva | 24h | Fixo, definido pelo backend — não editável aqui |
| Parcelamento máximo no crédito | _a definir_ | |
| Desconto à vista / PIX | _a definir_ | |
| Valor mínimo de pedido (se houver) | _a definir_ | |
| Frete / taxa de entrega (canal SITE) | _a definir_ | |
| Chave PIX da loja | _a definir_ | |
| Horário de atendimento humano | _a definir_ | |

---

*Football World Store — API Integration Reference — Documento de uso interno.*
*Atualizar sempre que o módulo Public do backend ganhar novos endpoints.*
