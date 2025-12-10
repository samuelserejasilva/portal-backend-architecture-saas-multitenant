# Modulo Webhooks - Documentacao Tecnica

**Versao**: 1.0.0
**Modulo**: `modules.webhooks`
**Package**: `com.auditoria.portalweb.modules.webhooks`
**Status**: Producao (92% completo)

---

## Visao Geral

Sistema completo para recepcao e envio de webhooks via HTTP. Implementa validacao HMAC-SHA256, rate limiting, retry com backoff exponencial, e processamento assincrono. Suporta multi-tenancy e arquitetura event-driven.

**Arquivos**: 33 classes Java
**Testes**: 3 classes de integracao (8 testes)
**Banco**: 3 tabelas com indices otimizados

---

## Arquitetura

### Estrutura de Pacotes

```
modules/webhooks/
├── domain/                    (3 entities)
│   ├── WebhookReceived.java
│   ├── WebhookSubscription.java
│   └── WebhookDelivery.java
├── repository/                (3 repositories)
├── internal/                  (10 services + handlers)
│   ├── WebhookSenderService.java
│   ├── WebhookReceiveService.java
│   ├── WebhookReceivedWorker.java
│   ├── WebhookRetryScheduler.java
│   ├── WebhookAdminService.java
│   ├── StripeWebhookHandler.java
│   ├── AsaasWebhookHandler.java
│   ├── IncomingWebhookHmacValidator.java
│   ├── WebhookSecurityProperties.java
│   └── WebhookRateLimitInterceptor.java
├── web/                       (5 controllers)
│   ├── WebhookReceiveController.java
│   ├── WebhookAdminSubscriptionController.java
│   ├── WebhookAdminDeliveryController.java
│   ├── WebhookAdminReceivedController.java
│   └── WebhookExceptionHandler.java
├── api/dto/                   (4 DTOs)
├── events/                    (4 event classes + listeners)
└── package-info.java
```

### Fluxo de Dados

**Recebimento (Incoming)**:
```
[Sistema Externo] --POST--> [Rate Limit] --HMAC--> [Controller] --Save--> [Database]
                                                                              |
                                                    [Worker a cada 30s] <-----+
                                                             |
                                        [Handler (Stripe/Asaas)] --Publish--> [Event Listeners]
```

**Envio (Outgoing)**:
```
[Admin API] --Create--> [WebhookDelivery (PENDING)]
                                  |
                      [Scheduler a cada 60s]
                                  |
                         [WebhookSenderService] --HTTP POST + HMAC--> [Sistema Externo]
                                  |
                          [Status: SENT/FAILED/EXHAUSTED]
                                  |
                      [Retry com backoff exponencial]
```

---

## Modelo de Dominio

### 1. WebhookReceived (Webhooks Recebidos)

**Tabela**: `webhook_received`

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `id` | Long | PK auto-incremento |
| `empresaId` | Integer | Multi-tenancy (opcional) |
| `source` | String(50) | Fonte do webhook (stripe, asaas) |
| `externalId` | String(255) | ID externo (para duplicacao) |
| `headers` | LONGTEXT | Headers HTTP serializados (JSON) |
| `payload` | LONGTEXT | Payload completo (JSON) |
| `processedStatus` | Enum | PENDING, PROCESSED, FAILED, DISCARDED |
| `receivedAt` | LocalDateTime | Timestamp de recepcao |
| `processedAt` | LocalDateTime | Timestamp de processamento |
| `errorMessage` | String(1000) | Mensagem de erro |

**Indices**:
- `(source, external_id)` - Deteccao de duplicatas
- `(processed_status)` - Query do worker

**Estados**:
- `PENDING` -> Aguardando processamento
- `PROCESSED` -> Processado com sucesso
- `FAILED` -> Falha no processamento
- `DISCARDED` -> Duplicado (source + externalId)

### 2. WebhookSubscription (Subscricoes)

**Tabela**: `webhook_subscription`

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `id` | Long | PK auto-incremento |
| `empresaId` | Integer | Multi-tenancy |
| `nome` | String(100) | Nome da subscricao |
| `eventType` | String(100) | Tipo de evento (payment.success) |
| `targetUrl` | String(500) | URL destino |
| `secretKey` | String(128) | Secret para HMAC |
| `status` | Enum | ACTIVE, PAUSED, DISABLED |
| `maxRetries` | Integer | Maximo de tentativas (default: 5) |
| `timeoutSeconds` | Integer | Timeout HTTP (default: 10s) |
| `createdAt` | LocalDateTime | Data de criacao |
| `updatedAt` | LocalDateTime | Ultima atualizacao |

**Constraint Unica**: `(empresa_id, event_type, target_url)`

### 3. WebhookDelivery (Entregas)

**Tabela**: `webhook_delivery`

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `id` | Long | PK auto-incremento |
| `empresaId` | Integer | Multi-tenancy |
| `subscriptionId` | Long | FK para subscription |
| `eventType` | String(100) | Tipo do evento |
| `payload` | LONGTEXT | JSON a ser enviado |
| `status` | Enum | PENDING, SENT, FAILED, EXHAUSTED |
| `attemptCount` | Integer | Numero de tentativas |
| `nextAttemptAt` | LocalDateTime | Proxima tentativa |
| `lastResponseStatus` | Integer | HTTP status (200, 500) |
| `lastResponseBody` | TEXT | Corpo da resposta (1000 chars) |
| `createdAt` | LocalDateTime | Data de criacao |
| `updatedAt` | LocalDateTime | Ultima atualizacao |

**Indices**:
- `(status, next_attempt_at)` - Query do scheduler
- `(empresa_id)` - Filtragem multi-tenant

**Estados**:
- `PENDING` -> Aguardando envio
- `SENT` -> Enviado com sucesso (HTTP 2xx)
- `FAILED` -> Falha temporaria (sera retentado)
- `EXHAUSTED` -> Esgotou tentativas (maximo atingido)

---

## API REST

### Endpoint Publico

**POST** `/api/v1/webhooks/receive/{source}`

Recebe webhooks de sistemas externos com validacao HMAC.

**Headers Obrigatorios**:
- `Content-Type: application/json`
- `X-Webhook-Signature: sha256=<hex>`

**Query Parameters** (opcionais):
- `empresaId` (Integer) - ID da empresa

**Request Body**: JSON livre (qualquer estrutura)

**Response** (202 Accepted):
```json
{
  "id": 123,
  "source": "stripe",
  "receivedAt": "2025-12-09T10:30:00.123456"
}
```

**Codigos de Status**:
- `202 ACCEPTED` - Webhook recebido e validado
- `400 BAD REQUEST` - JSON invalido ou source invalido
- `401 UNAUTHORIZED` - HMAC invalido
- `429 TOO MANY REQUESTS` - Rate limit excedido

**Rate Limiting**:
- 60 requisicoes/minuto por IP + source
- Algoritmo: janela deslizante
- Configuravel: `portalweb.webhooks.receiveRateLimitPerMinute`

**Validacao HMAC**:
```
HMAC-SHA256(payload, secret)
Header: X-Webhook-Signature: sha256=<hex>
```

**Exemplo (curl)**:
```bash
# Calcular signature
timestamp=$(date +%s)
signature=$(echo -n "${timestamp}.${payload}" | openssl dgst -sha256 -hmac "${secret}" | cut -d' ' -f2)

# Enviar webhook
curl -X POST "http://localhost:8080/api/v1/webhooks/receive/stripe" \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Signature: sha256=${signature}" \
  -d '{
    "id": "evt_123",
    "type": "payment_intent.succeeded",
    "data": { "object": { "id": "pi_123" } }
  }'
```

### Admin APIs

**Subscriptions**:
```
GET    /api/v1/admin/webhooks/subscriptions           Lista subscricoes
POST   /api/v1/admin/webhooks/subscriptions           Cria subscricao
PUT    /api/v1/admin/webhooks/subscriptions/{id}      Atualiza subscricao
```

**Request (POST/PUT)**:
```json
{
  "empresaId": 1,
  "nome": "Stripe Payment Success",
  "eventType": "payment.success",
  "targetUrl": "https://app.example.com/webhooks/stripe",
  "secretKey": "whsec_...",
  "maxRetries": 5,
  "timeoutSeconds": 10
}
```

**Deliveries**:
```
GET    /api/v1/admin/webhooks/deliveries              Lista entregas
POST   /api/v1/admin/webhooks/deliveries/{id}/retry   Retry manual
```

**Received**:
```
GET    /api/v1/admin/webhooks/received                Lista recebidos
POST   /api/v1/admin/webhooks/received/{id}/reprocess Reprocessar
```

---

## Seguranca

### HMAC-SHA256 Validation

**Incoming Webhooks** (IncomingWebhookHmacValidator):
- Algoritmo: HMAC-SHA256
- Comparacao: constant-time (`MessageDigest.isEqual()`)
- Formatos suportados: `sha256=<hex>`, `v1=<hex>`, `<hex>`
- Configuracao: `portalweb.webhooks.incoming-secrets.<source>=<key>`

**Outgoing Webhooks** (WebhookSenderService):
- Assina: `timestamp.payload`
- Header enviado: `X-Webhook-Signature: sha256=<hex>`
- Header timestamp: `X-Webhook-Timestamp: 1234567890`
- Secret: armazenado em `WebhookSubscription.secretKey`

### Rate Limiting

**WebhookRateLimitInterceptor**:
- Chave: `{IP}|{source}`
- Janela: 60 segundos (deslizante)
- Limite: 60 requisicoes (configuravel)
- Storage: In-memory `ConcurrentHashMap`
- Thread-safe: `synchronized` per key

**Configuracao**:
```properties
portalweb.webhooks.receiveRateLimitPerMinute=60
```

### Input Validation

**Source**:
- Obrigatorio
- Max 50 caracteres
- Normalizado: lowercase, trim

**Payload**:
- Deve ser JSON valido
- Sem limite explicito de tamanho (usa LONGTEXT)

**ExternalId**:
- Opcional
- Max 255 caracteres
- Usado para deteccao de duplicatas

---

## Processamento

### Worker (WebhookReceivedWorker)

**Agendamento**: A cada 30 segundos (configuravel)

**Configuracao**:
```properties
portalweb.webhooks.worker-interval-ms=30000
```

**Logica**:
1. Query: `findTop100ByProcessedStatusOrderByReceivedAtAsc(PENDING)`
2. Para cada webhook:
   - Busca handler por `source` (case-insensitive)
   - Se existe: executa `handler.handle(webhook)`
   - Se sucesso: marca `PROCESSED` + `processedAt`
   - Se erro: marca `FAILED` + `errorMessage`
   - Se nao existe handler: marca `DISCARDED` + "No handler found"
3. Salva todos no banco

**Batch Size**: 100 registros
**Ordenacao**: FIFO (por `receivedAt`)

### Handlers

**Interface**: `WebhookHandler`
```java
public interface WebhookHandler {
  String getSource();                           // Ex: "stripe", "asaas"
  void handle(WebhookReceived webhook) throws Exception;
}
```

**Implementacoes**:

1. **StripeWebhookHandler**
   - Source: `stripe`
   - Extrai: `type` e `data.object.id`
   - Publica: `IncomingStripeWebhookEvent`

2. **AsaasWebhookHandler**
   - Source: `asaas`
   - Extrai: `event` ou `type` e `id` ou `data.id`
   - Publica: `IncomingAsaasWebhookEvent`

**Event Listeners**:
```java
@EventListener
public void onStripeWebhook(IncomingStripeWebhookEvent event) {
  // Implementar logica de negocio
  // Ex: atualizar status de pagamento, enviar email
}

@EventListener
public void onAsaasWebhook(IncomingAsaasWebhookEvent event) {
  // Implementar logica de negocio
}
```

**Arquitetura Event-Driven**:
- Handlers publicam eventos via `ApplicationEventPublisher`
- Listeners sao descobertos automaticamente pelo Spring
- Desacoplamento: handlers nao conhecem consumidores

---

## Envio de Webhooks

### Retry Scheduler (WebhookRetryScheduler)

**Agendamento**: A cada 60 segundos (configuravel)

**Configuracao**:
```properties
portalweb.webhooks.retry-interval-ms=60000
```

**Logica**:
1. Query: `findTop100ByStatusInAndNextAttemptAtBefore(PENDING, FAILED)`
2. Para cada delivery: chama `WebhookSenderService.sendDelivery(id)`
3. Processa em lote (batch de 100)

### Sender Service (WebhookSenderService)

**Funcionalidades**:
- HTTP POST com `HttpClient` (Java 11+)
- HMAC-SHA256 signature do payload
- Timeout configuravel (default: 10s)
- Captura status + body da resposta
- Transacional (`@Transactional`)

**Headers Enviados**:
```
Content-Type: application/json
User-Agent: PortalAuditoria-Webhook/1.0
X-Webhook-Timestamp: 1234567890
X-Webhook-Signature: sha256=abc123...
```

**Logica de Retry**:

1. **Sucesso (HTTP 2xx)**:
   - Status: `SENT`
   - `nextAttemptAt`: `null`
   - Para de tentar

2. **Falha (HTTP 4xx/5xx ou Exception)**:
   - Incrementa `attemptCount`
   - Se `attemptCount >= maxRetries`: Status `EXHAUSTED`, para
   - Senao: Status `FAILED`, agenda proxima tentativa

**Backoff Exponencial**:
```
delaySeconds = 30 * 2^(attemptCount - 1)
max = 43200 (12 horas)

Tentativa 1: 30 segundos
Tentativa 2: 60 segundos (1 min)
Tentativa 3: 120 segundos (2 min)
Tentativa 4: 240 segundos (4 min)
Tentativa 5: 480 segundos (8 min)
Tentativa 6: 960 segundos (16 min)
Tentativa 7+: 43200 segundos (12h, maximo)
```

**Exemplo de Codigo**:
```java
// Linha 106-108 de WebhookSenderService.java
double seconds = 30 * Math.pow(2, Math.max(0, currentAttempt - 1));
long delaySeconds = (long) Math.min(seconds, 43200); // cap 12h
delivery.setNextAttemptAt(LocalDateTime.now(UTC).plusSeconds(delaySeconds));
```

---

## Configuracao

### Application Properties

```properties
# Secrets para validacao HMAC (incoming)
portalweb.webhooks.incoming-secrets.stripe=whsec_stripe_secret_123
portalweb.webhooks.incoming-secrets.asaas=asaas_webhook_key_456

# Rate limiting (requisicoes/minuto)
portalweb.webhooks.receiveRateLimitPerMinute=60

# Workers (intervalos em milissegundos)
portalweb.webhooks.worker-interval-ms=30000      # Worker processamento
portalweb.webhooks.retry-interval-ms=60000       # Scheduler retry
```

### Valores Padrao (se nao configurados)

| Propriedade | Valor Padrao |
|------------|--------------|
| `receiveRateLimitPerMinute` | 60 |
| `worker-interval-ms` | 30000 (30s) |
| `retry-interval-ms` | 60000 (60s) |
| `maxRetries` | 5 (por subscription) |
| `timeoutSeconds` | 10 (por subscription) |

---

## Banco de Dados

### Indices Criados

**webhook_received**:
```sql
CREATE INDEX idx_webhook_received_source_external_id
  ON webhook_received(source, external_id);

CREATE INDEX idx_webhook_received_status
  ON webhook_received(processed_status);
```

**webhook_delivery**:
```sql
CREATE INDEX idx_webhook_delivery_status_next
  ON webhook_delivery(status, next_attempt_at);

CREATE INDEX idx_webhook_delivery_empresa
  ON webhook_delivery(empresa_id);
```

**webhook_subscription**:
```sql
ALTER TABLE webhook_subscription
  ADD UNIQUE KEY uq_webhook_subscription_empresa_event_url
  (empresa_id, event_type, target_url);
```

### Tamanhos de Colunas

| Coluna | Tipo | Limite |
|--------|------|--------|
| `payload` | LONGTEXT | 4 GB (MySQL) |
| `headers` | LONGTEXT | 4 GB (MySQL) |
| `lastResponseBody` | TEXT | Truncado 1000 chars |
| `errorMessage` | String(1000) | Truncado 1000 chars |
| `targetUrl` | String(500) | 500 chars |
| `source` | String(50) | 50 chars |
| `externalId` | String(255) | 255 chars |

---

## Testes

### Classes de Teste (3)

**1. WebhookSenderServiceTest**
- Testa envio HTTP com HMAC
- Testa tratamento de erro
- Usa `MockWebServer` (okhttp3)
- Verifica backoff exponencial

**2. WebhookReceivedWorkerIT**
- Testa processamento com handler
- Testa descarte quando sem handler
- Verifica transicoes de estado

**3. WebhookReceiveControllerIT**
- Testa recepcao com HMAC valido (202)
- Testa rejeicao HMAC invalido (401)
- Testa rejeicao JSON invalido (400)

**Total**: 8 testes de integracao

**Infraestrutura**:
- Spring Boot Test
- H2 in-memory database
- MockMvc (controller)
- MockWebServer (HTTP)

---

## Padroes Implementados

### 1. Repository Pattern
Acesso ao banco encapsulado em repositories JPA.

### 2. Builder Pattern
Entities usam Lombok `@Builder` para construcao fluente.

### 3. DTO Pattern
Responses desacopladas das entities (WebhookReceiveResponse, etc).

### 4. Event-Driven Architecture
Handlers publicam eventos para listeners externos.

### 5. Scheduled Task Pattern
Workers executam periodicamente (`@Scheduled`).

### 6. Retry with Exponential Backoff
Retry progressivo para entregas falhadas.

### 7. Interceptor Pattern
Rate limiting via `HandlerInterceptor`.

### 8. Strategy Pattern
Handlers implementam interface comum, descobertos dinamicamente.

---

## Observabilidade

### Logs Implementados

**WebhookReceiveController**:
```java
log.info("Recebido webhook source={} empresaId={} tamanho={} bytes",
  source, empresaId, payload.length());
```

**WebhookReceivedWorker**:
```java
log.info("Processando {} webhooks pendentes", pending.size());
log.info("Processado webhook id={} source={} status={}",
  webhook.getId(), webhook.getSource(), webhook.getProcessedStatus());
```

**WebhookSenderService**:
```java
log.info("Enviando delivery id={} para {}", delivery.getId(), targetUrl);
log.info("Delivery id={} enviada com sucesso: HTTP {}", id, statusCode);
log.error("Erro ao enviar delivery id={}: {}", id, e.getMessage());
```

**WebhookRetryScheduler**:
```java
log.info("Scheduler encontrou {} deliveries para retry", deliveries.size());
```

### Recomendacoes

**Metricas (Micrometer)**:
- Counter: `webhooks.received{source}`
- Counter: `webhooks.sent{status}`
- Timer: `webhooks.processing.duration`
- Gauge: `webhooks.pending.count`

**Health Check**:
- Verificar backlog de PENDING
- Alertar se > 1000 webhooks pendentes

**Distributed Tracing**:
- Adicionar trace IDs nos logs
- Integrar com Sleuth/Zipkin

---

## Integracao

### Spring Modulith

```java
@ApplicationModule
package com.auditoria.portalweb.modules.webhooks;
```

**Isolamento**: Modulo nao declara dependencias externas.

**Integracao via Eventos**:
- Outros modulos podem ouvir `IncomingStripeWebhookEvent`
- Outros modulos podem ouvir `IncomingAsaasWebhookEvent`
- Loose coupling via Spring Events

**Exemplo de Listener Externo**:
```java
// Em outro modulo (users, payments, etc)
@Component
public class PaymentWebhookListener {

  @EventListener
  public void onStripePayment(IncomingStripeWebhookEvent event) {
    if ("payment_intent.succeeded".equals(event.eventType())) {
      // Atualizar status do pagamento
      paymentService.markAsPaid(event.objectId());
    }
  }
}
```

---

## Exemplos de Uso

### Configurar Webhook de Stripe

```java
WebhookSubscriptionRequest request = new WebhookSubscriptionRequest();
request.setEmpresaId(1);
request.setNome("Stripe - Pagamentos");
request.setEventType("payment_intent.succeeded");
request.setTargetUrl("https://api.empresa.com/webhooks/stripe-payment");
request.setSecretKey("whsec_...");
request.setMaxRetries(5);
request.setTimeoutSeconds(10);

POST /api/v1/admin/webhooks/subscriptions
Content-Type: application/json
{request}
```

### Receber Webhook do Stripe

**Stripe envia**:
```bash
POST /api/v1/webhooks/receive/stripe
X-Webhook-Signature: sha256=abc123...
Content-Type: application/json

{
  "id": "evt_123",
  "type": "payment_intent.succeeded",
  "data": {
    "object": {
      "id": "pi_123",
      "amount": 5000
    }
  }
}
```

**Sistema processa**:
1. Rate limit OK (< 60/min)
2. HMAC valido (compara signature)
3. Salva no banco (PENDING)
4. Retorna 202 ACCEPTED

**Worker processa (30s depois)**:
1. Busca webhook PENDING
2. Identifica handler: `StripeWebhookHandler`
3. Handler extrai tipo e ID
4. Publica evento: `IncomingStripeWebhookEvent`
5. Marca webhook como PROCESSED

**Listener externo consome**:
```java
@EventListener
public void onStripePayment(IncomingStripeWebhookEvent event) {
  log.info("Pagamento aprovado: {}", event.objectId());
  // Atualizar banco de dados, enviar email, etc
}
```

### Enviar Webhook para Cliente

**Admin cria delivery**:
```java
WebhookDelivery delivery = new WebhookDelivery();
delivery.setEmpresaId(1);
delivery.setSubscriptionId(123L); // FK
delivery.setEventType("order.completed");
delivery.setPayload("{\"orderId\": 456, \"total\": 100.50}");
delivery.setStatus(Status.PENDING);
repository.save(delivery);
```

**Scheduler processa (60s depois)**:
1. Busca deliveries PENDING/FAILED
2. WebhookSenderService envia HTTP POST
3. Assina payload com HMAC
4. Se sucesso: marca SENT
5. Se erro: marca FAILED + agenda retry
6. Retry com backoff: 30s, 1min, 2min, 4min...

### Forcar Retry Manual

```bash
POST /api/v1/admin/webhooks/deliveries/123/retry
```

Marca delivery como PENDING e `nextAttemptAt = now()`, forcando reprocessamento imediato.

### Reprocessar Webhook Recebido

```bash
POST /api/v1/admin/webhooks/received/456/reprocess
```

Marca webhook como PENDING, limpando erro. Worker reprocessara na proxima execucao.

---

## Limitacoes Conhecidas

1. **Payload Size**: Sem limite explicito (usa LONGTEXT). Recomendado adicionar validacao de 1-5 MB.

2. **Rate Limit Storage**: In-memory (perde ao reiniciar). Para producao distribuida, considerar Redis.

3. **Dead Letter Queue**: Webhooks EXHAUSTED ficam no banco. Considerar movimentacao para tabela de DLQ.

4. **Metrics**: Sem metricas exportadas (Micrometer). Adicionar para monitoramento.

---

## Referencias Tecnicas

- **Java HttpClient**: https://docs.oracle.com/en/java/javase/11/docs/api/java.net.http/java/net/http/HttpClient.html
- **HMAC-SHA256**: https://datatracker.ietf.org/doc/html/rfc2104
- **Webhook Best Practices**: https://webhooks.fyi/best-practices/
- **Exponential Backoff**: https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/
- **Spring Events**: https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html
- **Spring Modulith**: https://docs.spring.io/spring-modulith/reference/

---

**Autor**: Equipe Backend Portal Auditoria
**Versao**: 1.0.0 (Producao)
