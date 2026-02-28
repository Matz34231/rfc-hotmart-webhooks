# RFC --- Processamento de Webhooks Hotmart

**AvaliaFácil \| Prisma Tech**

------------------------------------------------------------------------

## 1. De-Para de Dados (JSON → Banco)

  ----------------------------------------------------------------------------------------------
  Campo JSON                            Coluna Banco                 Regra Aplicada
  ------------------------------------- ---------------------------- ---------------------------
  `id`                                  `external_event_id`          Identificador único do
                                                                     evento

  `data.purchase.transaction`           `external_transaction_id`    Identificador único da
                                                                     transação

  `data.subscription.subscriber.code`   `external_subscription_id`   Identificador único da
                                                                     assinatura

  `data.subscription.status`            `status`                     Mapear ACTIVE, OVERDUE,
                                                                     CANCELLED

  `data.purchase.status`                `purchase_status`            APPROVED ou COMPLETED
                                                                     indicam pagamento
                                                                     confirmado

  `data.purchase.approved_date`         `approved_at`                Converter timestamp (ms)
                                                                     para datetime

  `data.purchase.date_next_charge`      `access_ends_at`             Converter timestamp (ms)
                                                                     para datetime

  `data.subscription.plan.name`         `plan_type`                  Determinar mensal vs anual

  `data.purchase.recurrence_number`     `recurrence_number`          Controle de renovação

  `data.buyer.email`                    `user_email`                 Identificação do usuário

  Payload completo                      `raw_webhook_data`           Armazenado como JSONB para
                                                                     auditoria
  ----------------------------------------------------------------------------------------------

------------------------------------------------------------------------

## Conversão de Timestamp

``` pseudo
datetime = timestamp_ms / 1000
```

------------------------------------------------------------------------

## 2. Lógica de Negócio

### Recebimento do Webhook

1.  Validar `hottok`
2.  Verificar idempotência usando `external_event_id`
3.  Persistir `raw_webhook_data`
4.  Processar regra conforme o `event`

------------------------------------------------------------------------

### Liberação de Acesso

Eventos: - `PURCHASE_APPROVED` - `PURCHASE_COMPLETED`

``` pseudo
if event in ["PURCHASE_APPROVED", "PURCHASE_COMPLETED"]:
    criar ou atualizar assinatura
    status = ACTIVE
    access_ends_at = date_next_charge
```

------------------------------------------------------------------------

### Cancelamento

``` pseudo
event = SUBSCRIPTION_CANCELLATION
status = CANCELLED
manter access_ends_at
```

------------------------------------------------------------------------

### Reembolso

``` pseudo
event = REFUNDED
status = REFUNDED
access_ends_at = now()
```

------------------------------------------------------------------------

### Regra Final de Acesso

``` pseudo
if status == ACTIVE and now() <= access_ends_at:
    acesso = LIBERADO
else:
    acesso = BLOQUEADO
```

------------------------------------------------------------------------

## 3. Fluxo de Renovação

``` pseudo
if recurrence_number > recurrence_number_anterior:
    atualizar access_ends_at
```

Controle de idempotência:

``` pseudo
if external_event_id já existe:
    ignorar evento
```

------------------------------------------------------------------------

## 4. Estratégia de Testes

Endpoint:

POST http://localhost:8000/api/webhook/hotmart

### Cenários Testados

1.  Compra Aprovada → Criação de assinatura ACTIVE
2.  Renovação → Atualização de access_ends_at
3.  Cancelamento → Status CANCELLED mantendo período
4.  Reembolso → Bloqueio imediato
5.  Idempotência → Reenvio do mesmo id não duplica dados

------------------------------------------------------------------------

## 5. Estratégia de Testes (Simulação Completa)

### Endpoint

POST http://localhost:8000/api/webhook/hotmart

### Payload Base

``` json
{
  "id": "evt_123456",
  "event": "PURCHASE_APPROVED",
  "data": {
    "purchase": {
      "transaction": "HP123456789",
      "status": "APPROVED",
      "approved_date": 1700000000000,
      "date_next_charge": 1702592000000,
      "recurrence_number": 1
    },
    "subscription": {
      "subscriber": { "code": "SUB123456" },
      "status": "ACTIVE",
      "plan": { "name": "Plano Mensal" }
    },
    "buyer": { "email": "cliente@email.com" }
  }
}
```

------------------------------------------------------------------------

### Cenários

#### 1) Compra Aprovada

-   Criar assinatura
-   Status ACTIVE
-   Liberar acesso

#### 2) Renovação

-   recurrence_number = 2
-   Atualizar access_ends_at

#### 3) Cancelamento

-   event = SUBSCRIPTION_CANCELLATION
-   Manter acesso até fim do período

#### 4) Reembolso

-   event = REFUNDED
-   Bloqueio imediato

#### 5) Idempotência

-   Reenviar mesmo id
-   Evento ignorado

------------------------------------------------------------------------

## 6. Mentalidade AI-First

### 6.1 Análise de Payloads

A IA foi utilizada para: - Fazer a leitura da estrutura dos JSONs reais
da Hotmart. - Detectar diferenças entre eventos como `PURCHASE_APPROVED`
e `PURCHASE_COMPLETED`.

Isso eliminou a necessidade de leitura extensa de documentação estática,
acelerando a modelagem de dados.

------------------------------------------------------------------------

### 6.2 Mapeamento de Cenários e Edge Cases

Com auxílio de IA foi possível:

-   Identificar distinção entre nova assinatura e renovação (via
    `recurrence_number`).
-   Mapear cenários de cancelamento e reembolso.
-   Definir estratégia de idempotência baseada em `external_event_id`.
-   Antecipar possíveis inconsistências de status entre
    `subscription.status` e `purchase.status`.

------------------------------------------------------------------------

### 6.3 Geração de Arquitetura

A IA auxiliou na:

-   Definição da estrutura da entidade `HotmartSubscription`.
-   Organização do fluxo do webhook em etapas (validação, idempotência,
    persistência, regra de negócio).
-   Proposição de índice único para `external_event_id`.
-   Separação clara entre regra de pagamento e regra de acesso.

------------------------------------------------------------------------

### 6.4 Impacto na Produtividade

A utilização de IA permitiu:

-   Redução significativa do tempo de análise dos payloads.
-   Maior clareza na definição das regras de negócio.
-   Documentação técnica mais estruturada e completa.
-   Melhor entendimento do código.

------------------------------------------------------------------------

## 7. Conclusão

-   Liberação imediata após pagamento confirmado\
-   Acesso mantido até o fim do período pago em cancelamento\
-   Revogação imediata em reembolso\
-   Tratamento correto de renovações\
-   Auditoria via `raw_webhook_data`\
-   Idempotência garantida\
-   Uso estratégico de IA como acelerador de arquitetura
