# Fidexa Partner API

API para parceiros enviarem mensagens de WhatsApp e gerenciarem clientes na
plataforma Fidexa. Esta referência cobre **autenticação**, **clientes** e
**mensageria WhatsApp**.

- **Base URL:** `https://api.fidexa.com.br`
- **Formato:** JSON (`Content-Type: application/json`)
- **Convenção de chaves:** **`snake_case`** no corpo (request e response)
- **Auth:** `Bearer <JWT>` em todos os endpoints, exceto `auth/*`

---

## Sumário

- [Autenticação](#autenticação)
  - [`POST /auth/sign-in`](#post-authsign-in)
  - [`POST /auth/refresh-token`](#post-authrefresh-token)
- [Clientes](#clientes)
  - [`POST /customers`](#post-customers)
  - [`POST /customers/batch`](#post-customersbatch)
- [Mensageria WhatsApp](#mensageria-whatsapp)
  - [`POST /whatsapp/message`](#post-whatsappmessage)
  - [`POST /whatsapp/message-text`](#post-whatsappmessage-text)
  - [`POST /whatsapp/message-pix/create`](#post-whatsappmessage-pixcreate)
  - [`POST /whatsapp/message-pix/paid`](#post-whatsappmessage-pixpaid)
- [Formatos e convenções](#formatos-e-convenções)
- [Tratamento de erros](#tratamento-de-erros)

---

## Autenticação

A API usa **JWT (Bearer)**. Faça login em `auth/sign-in` para obter um
`access_token` (curta duração) e um `refresh_token`. Quando o `access_token`
expirar, use `auth/refresh-token` para obter um novo sem refazer o login.

Envie o token em **todas** as chamadas protegidas:

```
Authorization: Bearer <access_token>
```

### `POST /auth/sign-in`

Autentica o parceiro e retorna os tokens. **Não requer** autenticação.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|:---:|---|
| `phone` | string | sim | Telefone do usuário, com '+' e DDI (ex: `+5519999999999`). |
| `password` | string | sim | Senha do usuário. |

**Request**
```json
{
  "phone": "+5519999999999",
  "password": "sua-senha"
}
```

**`200 OK`**
```json
{
  "access_token": "eyJhbGciOi...",
  "features": ["send_message", "..."],
  "refresh_token": "8f3c1b2a-4d5e-6f70-8190-a1b2c3d4e5f6"
}
```

### `POST /auth/refresh-token`

Renova o `access_token` a partir de um par válido. **Não requer** autenticação.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|:---:|---|
| `access_token` | string | sim | O `access_token` atual (mesmo expirado). |
| `refresh_token` | uuid | sim | O `refresh_token` recebido no login. |

**Request**
```json
{
  "access_token": "eyJhbGciOi...",
  "refresh_token": "8f3c1b2a-4d5e-6f70-8190-a1b2c3d4e5f6"
}
```

**`201 Created`** — mesmo corpo do `sign-in` (novo `access_token` e
`refresh_token`).

---

## Clientes

### `POST /customers`

Cadastra um cliente em uma empresa do parceiro. 🔒 **Requer Bearer.**

A empresa (`company_id`) precisa pertencer ao seu grupo. Telefones são
normalizados para `+55DDD9XXXXXXXX` (aceitam entrada com ou sem máscara) e o
`cpf_cnpj` é validado e gravado apenas com dígitos. O `identifier` é único por
empresa.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|:---:|---|
| `company_id` | uuid | sim | Empresa à qual o cliente pertence. |
| `identifier` | string | sim | Código do cliente, único por empresa. |
| `full_name` | string | sim | Nome completo. |
| `email` | string | não | E-mail do cliente. |
| `cpf_cnpj` | string | sim | CPF ou CNPJ válido (com ou sem pontuação). |
| `cell_phone1` | string | sim | Telefone com DDD (ex: `19982358635`). |
| `cell_phone2` | string | não | Celular com DDD. |
| `date_of_birth` | date | não | Data de nascimento (`yyyy-MM-dd`). |

**Request**
```json
{
  "company_id": "FBB64ED3-8FD1-4B5B-B53A-00002F4A89B8",
  "identifier": "C-1010",
  "full_name": "LEONARDO AZEVEDO",
  "email": "leonardo.azevedo@fidexa.com.br",
  "cpf_cnpj": "12332112333",
  "cell_phone1": "19982358635",
  "cell_phone2": "19982358635",
  "date_of_birth": "1990-01-01"
}
```

**`201 Created`**
```json
{
  "id": "03EC2B3E-0025-4680-8F3B-00002E883F79",
  "company_id": "FBB64ED3-8FD1-4B5B-B53A-00002F4A89B8",
  "identifier": "C-1010",
  "full_name": "LEONARDO AZEVEDO",
  "email": "leonardo.azevedo@fidexa.com.br",
  "cpf_cnpj": "12332112333",
  "cell_phone1": "+5519982358635",
  "cell_phone2": "+5519982358635",
  "date_of_birth": "1990-01-01"
}
```

| Status | Quando |
|---|---|
| `400` | Campo obrigatório ausente, telefone ou CPF/CNPJ inválido. |
| `404` | `company_id` não encontrado para o seu grupo. |
| `409` | Já existe um cliente com este `identifier` na empresa. |

### `POST /customers/batch`

Cadastra **até 1000** clientes de uma vez, todos da mesma empresa. 🔒 **Requer
Bearer.**

Cada item é processado de forma **independente**: itens válidos são criados e
itens inválidos retornam erro, **sem abortar o lote**.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|:---:|---|
| `company_id` | uuid | sim | Empresa de todos os clientes do lote. |
| `customers` | array | sim | 1 a 1000 itens. Mesmos campos do `POST /customers`, **sem** `company_id`. |

**Request**
```json
{
  "company_id": "FBB64ED3-8FD1-4B5B-B53A-00002F4A89B8",
  "customers": [
    {
      "identifier": "C-1010",
      "full_name": "LEONARDO AZEVEDO",
      "email": "leonardo.azevedo@fidexa.com.br",
      "cpf_cnpj": "12332112333",
      "cell_phone1": "19982358635",
      "cell_phone2": "19982358635",
      "date_of_birth": "1990-01-01"
    },
    {
      "identifier": "C-1011",
      "full_name": "MARIA SILVA",
      "cpf_cnpj": "98765432100",
      "cell_phone1": "19999998888"
    }
  ]
}
```

**Response**
```json
{
  "summary": { "total": 2, "success": 1, "failed": 1 },
  "results": [
    {
      "index": 0,
      "status": "success",
      "data": {
        "id": "03EC2B3E-0025-4680-8F3B-00002E883F79",
        "company_id": "FBB64ED3-8FD1-4B5B-B53A-00002F4A89B8",
        "identifier": "C-1010",
        "full_name": "LEONARDO AZEVEDO",
        "email": "leonardo.azevedo@fidexa.com.br",
        "cpf_cnpj": "12332112333",
        "cell_phone1": "+5519982358635",
        "cell_phone2": "+5519982358635",
        "date_of_birth": "1990-01-01"
      }
    },
    {
      "index": 1,
      "status": "error",
      "identifier": "C-1011",
      "error": { "code": "INVALID_CPF_CNPJ", "message": "CPF/CNPJ inválido" }
    }
  ]
}
```

**Status HTTP** (reflete o resultado agregado):

| Status | Quando |
|---|---|
| `201` | Todos os itens foram cadastrados. |
| `207` | Lote parcial — parte com sucesso, parte com erro. |
| `404` | Todos os itens falharam, ou `company_id` não encontrado. |
| `400` | Requisição inválida (lista vazia ou acima de 1000). |

**Códigos de erro por item** (`results[].error.code`):
`INVALID_IDENTIFIER`, `INVALID_FULL_NAME`, `INVALID_CPF_CNPJ`,
`INVALID_CELL_PHONE1`, `INVALID_CELL_PHONE2`, `DUPLICATE_IDENTIFIER`.

---

## Mensageria WhatsApp

Todos os endpoints abaixo 🔒 **requerem Bearer** e usam telefones no formato
**internacional com `+` e DDI** (ex: `+5519982358635`).

> O `sender_phone` precisa ser o número cadastrado para o seu grupo na Fidexa.

### `POST /whatsapp/message`

Envia uma mensagem baseada em **template aprovado** (categoria `UTILITY`).

| Campo | Tipo | Obrigatório | Descrição |
|---|---|:---:|---|
| `sender_phone` | string | sim | Número de envio do seu grupo (`+DDI...`). |
| `recipient_phone` | string | sim | Número do destinatário (`+DDI...`). |
| `template_name` | string | sim | Nome do template aprovado no Meta. |
| `variables` | object | sim | Pares chave/valor das variáveis do template. |
| `callback` | string | não | Conteúdo (link/texto) a ser disparado como follow-up. |

**Request**
```json
{
  "sender_phone": "+5519999999999",
  "recipient_phone": "+5519982358635",
  "template_name": "aviso_fatura",
  "variables": {
    "nome": "Leonardo",
    "valor": "R$ 150,00"
  },
  "callback": "https://app.fidexa.com.br/fatura/123"
}
```

**`200 OK`**
```json
{ "message_id": "03EC2B3E-0025-4680-8F3B-00002E883F79" }
```

| Status | Quando |
|---|---|
| `400` | Telefones inválidos, `variables` divergentes do template ou template não `APPROVED`/`UTILITY`. |
| `404` | Número de envio, WhatsApp Business ou template não encontrado. |

### `POST /whatsapp/message-text`

Envia uma **mensagem de texto livre** (requer janela de atendimento aberta).

| Campo | Tipo | Obrigatório | Descrição |
|---|---|:---:|---|
| `sender_phone` | string | sim | Número de envio do seu grupo (`+DDI...`). |
| `recipient_phone` | string | sim | Número do destinatário (`+DDI...`). |
| `message` | string | sim | Texto da mensagem. |

**Request**
```json
{
  "sender_phone": "+5519999999999",
  "recipient_phone": "+5519982358635",
  "message": "Olá! Sua fatura já está disponível."
}
```

**`200 OK`**
```json
{ "message_id": "03EC2B3E-0025-4680-8F3B-00002E883F79" }
```

### `POST /whatsapp/message-pix/create`

Envia um template com **cobrança PIX** (código dinâmico). Retorna também o
`callback_message_id`, usado para confirmar o pagamento depois.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|:---:|---|
| `sender_phone` | string | sim | Número de envio do seu grupo (`+DDI...`). |
| `recipient_phone` | string | sim | Número do destinatário (`+DDI...`). |
| `template_name` | string | sim | Nome do template aprovado. |
| `text_body` | string | sim | Texto que acompanha a cobrança. |
| `payment_id` | string | sim | Seu identificador do pagamento. |
| `variables` | object | sim | Variáveis do template. |
| `pix` | object | sim | Dados do PIX (ver abaixo). |

**`pix`**
| Campo | Tipo | Descrição |
|---|---|---|
| `payment_settings[].pix_dynamic_code` | object | `code`, `merchant_name`, `key`, `key_type`. |
| `items[]` | array | `retailer_id`, `name`, `amount.value` (centavos), `quantity`. |
| `subtotal.value` | long | Subtotal em **centavos**. |

**Request**
```json
{
  "sender_phone": "+5519999999999",
  "recipient_phone": "+5519982358635",
  "template_name": "cobranca_pix",
  "text_body": "Pague sua fatura via PIX",
  "payment_id": "PAY-123",
  "variables": { "nome": "Leonardo" },
  "pix": {
    "payment_settings": [
      {
        "pix_dynamic_code": {
          "code": "00020126...",
          "merchant_name": "FIDEXA",
          "key": "pix@fidexa.com.br",
          "key_type": "EMAIL"
        }
      }
    ],
    "items": [
      {
        "retailer_id": "SKU-1",
        "name": "Fatura Janeiro",
        "amount": { "value": 15000 },
        "quantity": 1
      }
    ],
    "subtotal": { "value": 15000 }
  }
}
```

**`200 OK`**
```json
{
  "message_id": "03EC2B3E-0025-4680-8F3B-00002E883F79",
  "callback_message_id": "9A1B2C3D-4E5F-6071-8293-A4B5C6D7E8F9"
}
```

### `POST /whatsapp/message-pix/paid`

Marca uma cobrança PIX como **paga**, usando o `callback_message_id` retornado
no `create`.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|:---:|---|
| `callback_message_id` | uuid | sim | Id retornado em `message-pix/create`. |

**Request**
```json
{ "callback_message_id": "9A1B2C3D-4E5F-6071-8293-A4B5C6D7E8F9" }
```

**`200 OK`**
```json
{ "message_id": "03EC2B3E-0025-4680-8F3B-00002E883F79" }
```

---

## Formatos e convenções

| Tema | Regra |
|---|---|
| Corpo JSON | Sempre `snake_case`. |
| Telefones (WhatsApp) | Formato internacional com `+` e DDI: `+5519982358635`. |
| Telefones (clientes) | Com DDD; normalizados e gravados como `+55DDD9XXXXXXXX`. |
| CPF/CNPJ | Com ou sem pontuação; validados e gravados só com dígitos. |
| Datas | `date_of_birth`: `yyyy-MM-dd`. |
| Valores PIX | Em **centavos** (long). Ex.: `15000` = R$ 150,00. |
| Autorização | `Authorization: Bearer <access_token>` (exceto `auth/*`). |

---

## Tratamento de erros

Erros seguem um formato consistente:

```json
{
  "name": "ErrorOnValidationException",
  "message": [
    "`cell_phone1` inválido. Informe DDD + número (ex: 19982358635)."
  ],
  "action": "Valide os campos obrigatórios.",
  "status_code": 400
}
```

| Campo | Descrição |
|---|---|
| `name` | Tipo do erro (ex.: `ErrorOnValidationException`, `NotFoundException`). |
| `message` | **Lista** de mensagens descritivas. |
| `action` | Orientação do que fazer para corrigir. |
| `status_code` | Código HTTP. |

**Status mais comuns**

| Status | Significado |
|---|---|
| `400 Bad Request` | Dados inválidos ou faltando. Veja `message`. |
| `401 Unauthorized` | Token ausente, inválido ou expirado — renove em `auth/refresh-token`. |
| `404 Not Found` | Recurso (empresa, template, número) não encontrado. |
| `409 Conflict` | Conflito de unicidade (ex.: `identifier` já existe). |

> Para lotes (`/customers/batch`), erros de **itens** não usam esse formato:
> eles vêm dentro de `results[].error` com `code`/`message`. O envelope acima
> vale apenas para falhas da requisição inteira.
