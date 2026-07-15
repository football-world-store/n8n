---
index_id: fws-master-index
version: 1.0
type: inverted-index
covers: [fws-central, fws-vendas, fws-reservas, fws-suporte, fws-financeiro]
description: Índice invertido mestre — mapeia cada termo, sinônimo e intenção ao agente e seção responsável
---

# Football World Store — Índice Invertido Mestre

Uso interno | Pipeline de roteamento N8N | Versão 1.0

> **Como usar:** Detecte o termo ou intenção na mensagem do cliente → consulte a tabela
> correspondente → despache para o agente indicado.

> Para o contrato exato dos endpoints que os agentes de Vendas e Reservas consomem, ver
> [api-endpoints.md](api-endpoints.md) — não é um agente, é referência de integração.

---

## ROTEAMENTO RÁPIDO POR PALAVRA-CHAVE

Lookup de primeiro nível. Se o termo aparecer na mensagem → rotear diretamente.

| Termo / Intenção detectada | Agente destino | Arquivo |
| --- | --- | --- |
| camisa, short, meião, agasalho, chuteira, acessório, time, tamanho, quanto custa, preço, catálogo, novidade, lançamento | `fws-vendas` | [fws-vendas.md](fws-vendas.md) |
| separar, guardar, reservar, segura pra mim, até quando fica reservado, cancelar reserva | `fws-reservas` | [fws-reservas.md](fws-reservas.md) |
| meu pedido, cadê minha encomenda, quero trocar, quero devolver, atualizar cadastro, comprei e não chegou | `fws-suporte` | [fws-suporte.md](fws-suporte.md) |
| pix, cartão, parcelar, débito, crédito, dinheiro, desconto, forma de pagamento | `fws-financeiro` | [fws-financeiro.md](fws-financeiro.md) |
| oi, olá, bom dia, preciso de ajuda, quero saber, como funciona | `fws-central` | [fws-central.md](fws-central.md) |

---

## 1. ÍNDICE POR AGENTE

### 1.1 fws-vendas

**Arquivo:** [fws-vendas.md](fws-vendas.md)

| Termo / Sinônimo | Seção | Conceito canônico |
| --- | --- | --- |
| camisa, short, meião, agasalho, chuteira, calçado, acessório | §2 Categorias de produto | categoria de produto |
| tem camisa do, tem no meu tamanho, tá disponível | §3 Regra crítica de estoque | consulta de estoque em tempo real |
| quanto custa, qual o preço, valor | §4 Como responder / §2 | preço de venda (público, pode informar) |
| catálogo, o que vocês vendem, lançamento, chegou, novidade | §4 Como responder | catálogo geral |
| separa pra mim, guarda esse | §5 Quando rotear para RESERVAS | handoff para reserva |
| como eu pago, vou levar mesmo | §5 Quando rotear para FINANCEIRO | handoff para pagamento |
| time internacional, camisa retrô, numeração de chuteira | §6 FAQ | perguntas frequentes de catálogo |

---

### 1.2 fws-reservas

**Arquivo:** [fws-reservas.md](fws-reservas.md)

| Termo / Sinônimo | Seção | Conceito canônico |
| --- | --- | --- |
| reservar, separar, guardar, segurar | §1 / §5 Fluxo | criação de reserva |
| até quando fica reservado, prazo | §2 Regra de prazo | validade fixa de 24h (definida pelo backend) |
| nome, whatsapp, e-mail, observação | §3 Dados necessários | dados da reserva |
| minha reserva foi confirmada?, e a reserva? | §6 Depois de criar a reserva | sem endpoint de consulta — escalar se demorar |
| cancelar reserva | §7 Cancelamento | sem endpoint de cancelamento — sempre ROUTE: HUMANO |
| avise quando chegar, me avisa quando tiver | §8 Lista de espera | endpoint futuro (proposta em api-endpoints.md) — hoje é coleta manual + escalação |

---

### 1.3 fws-suporte

**Arquivo:** [fws-suporte.md](fws-suporte.md)

| Termo / Sinônimo | Seção | Conceito canônico |
| --- | --- | --- |
| meu pedido, cadê minha encomenda | §2 Status de pedido | sem endpoint público — coletar e escalar |
| quero trocar, quero devolver | §3 Troca e devolução | escalação manual para equipe |
| atualizar meus dados, meu cadastro | §4 Cadastro e preferências | sem endpoint público — coletar e escalar |
| não resolvido após 3 mensagens | §5 Regra das 3 trocas | escalação para humano |

---

### 1.4 fws-financeiro

**Arquivo:** [fws-financeiro.md](fws-financeiro.md)

| Termo / Sinônimo | Seção | Conceito canônico |
| --- | --- | --- |
| pix, débito, crédito, dinheiro | §2 Formas de pagamento | formas de pagamento aceitas |
| parcelar, parcelado, quantas vezes | §3 Parcelamento e desconto | política de parcelamento (variável) |
| desconto, tem desconto | §3 Parcelamento e desconto | desconto — nunca afirmar sem confirmação |
| como eu pago, fechar pedido | §4 Fluxo de fechamento | fechamento de pagamento |
| pagar na entrega | §5 FAQ | pagamento por canal |

---

### 1.5 fws-central

**Arquivo:** [fws-central.md](fws-central.md)

| Sinal de roteamento | Destino |
| --- | --- |
| VENDAS | fws-vendas |
| RESERVAS | fws-reservas |
| SUPORTE | fws-suporte |
| FINANCEIRO | fws-financeiro |
| HUMANO | equipe-loja |
| CONTINUAR | mesmo agente |

---

## 2. ROTEAMENTO POR ENTIDADE DETECTADA

Quando o cliente envia diretamente uma entidade reconhecível:

| Entidade detectada | Formato / Padrão | Agente |
| --- | --- | --- |
| Nome de time/clube | texto livre (ex. "Flamengo", "Brasil", "Real Madrid") | fws-vendas |
| Tamanho | P, M, G, GG, numeração de calçado | fws-vendas |
| Número de pedido/venda | número inteiro | fws-suporte |
| E-mail | `[^@]+@[^@]+\.[^@]+` (contexto de cadastro) | fws-suporte |
| Telefone/WhatsApp | `\(?\d{2}\)?\s?\d{4,5}-?\d{4}` | fws-reservas / fws-suporte |

---

## 3. ROTEAMENTO POR PERFIL DO CLIENTE

| Vocabulário detectado | Perfil inferido | Ajuste de resposta |
| --- | --- | --- |
| "vocês têm coisa de futebol?", "quero ver umas camisas" | Visitante navegando | Descoberta → fws-vendas |
| "quero a camisa do [time] tamanho [M]" | Comprador decidido | Rotear direto para fws-vendas |
| "guarda uma pra eu buscar" | Cliente com reserva em mente | fws-reservas |
| "meu pedido", "comprei e não chegou" | Cliente com pedido em andamento | fws-suporte |
| "como eu pago", "vocês parcelam" | Cliente fechando pagamento | fws-financeiro |

---

## 4. MAPA DE ESCALAÇÃO

Quando nenhum agente consegue resolver, escalar conforme:

| Situação | Escalar para |
| --- | --- |
| Status de pedido | fws-suporte → equipe-loja (sem endpoint público) |
| Troca / devolução | fws-suporte → equipe-loja |
| Cadastro / atualização de dados | fws-suporte → equipe-loja (sem endpoint público) |
| Reclamação | fws-central / fws-suporte → equipe-loja |
| Desconto especial / negociação | fws-financeiro → equipe-loja |
| Problema no pagamento | fws-financeiro → equipe-loja |
| Estoque não confirmável (falha na consulta) | fws-vendas → equipe-loja (se cliente com pressa) |
| Confirmar, cancelar ou consultar reserva já criada | fws-reservas → equipe-loja (sem endpoint público) |
| Avise quando chegar (produto zerado) | fws-reservas → equipe-loja (endpoint futuro, ver api-endpoints.md) |

---

## 5. CLUSTERS SEMÂNTICOS (conceitos que co-ocorrem)

**Cluster: Descoberta de produto**
Time do coração → Categoria → Tamanho → Preço → Disponibilidade

**Cluster: Fechamento de venda**
Disponibilidade confirmada → Reserva ou Pagamento → Forma de pagamento → Confirmação

**Cluster: Reserva**
Produto + tamanho → Nome + WhatsApp → Prazo fixo de 24h → Confirmação manual pela loja (sem lookup pelo bot)

**Cluster: Pós-venda**
Status do pedido → Troca/Devolução → Escalação humana (sem endpoint público hoje)

**Cluster: Relacionamento com cliente**
Time do coração → Tamanhos preferidos → Conversa de personalização (não persistida via API)

---

## 6. TERMOS QUE NUNCA DEVEM SER RESPONDIDOS DIRETAMENTE

Estes termos devem acionar deflexão, consulta em tempo real ou escalação:

| Termo | Agente que NÃO responde de cabeça | Ação correta |
| --- | --- | --- |
| Disponibilidade / quantidade em estoque | fws-vendas | Consultar "Consultar Produtos" antes de afirmar |
| Status de pedido / histórico de compra | fws-suporte | Não existe endpoint — coletar e escalar |
| Aprovar troca/devolução | fws-suporte | Escalar equipe da loja |
| Desconto especial fora da tabela de valores | fws-financeiro | Escalar equipe da loja |
| Prazo de reserva diferente de 24h | fws-reservas | Prazo é fixo pelo backend — nunca prometer outro |
| Confirmar/cancelar reserva já criada | fws-reservas | Não existe endpoint — escalar equipe da loja |
| Dados de outro cliente | qualquer agente | Nunca compartilhar |

---

*Football World Store — Master Index — Documento de uso interno.*
*Atualizar sempre que um agente for criado, removido ou tiver seu escopo alterado.*
