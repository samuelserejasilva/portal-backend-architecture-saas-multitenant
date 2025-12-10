# Módulo Webhooks

**Subsistema de alta disponibilidade para orquestração de eventos (Inbound/Outbound).**

Projetado para operar em escala (Enterprise Ready), este módulo gerencia o ciclo de vida completo de integrações via webhooks, garantindo segurança, resiliência e observabilidade em um ambiente distribuído.

- **Arquitetura:** Event-Driven (Spring Modulith)
- **Segurança:** Validação HMAC-SHA256 e Rate Limiting Distribuído (Redis)
- **Resiliência:** Retry Exponencial, Dead Letter Queue (DLQ) e Fallback automático
- **Observabilidade:** Métricas em tempo real (Micrometer) e Health Checks de Backlog

---

## Visao Geral

Sistema completo para recepcao e envio de webhooks via HTTP. Implementa validacao HMAC-SHA256, rate limiting, retry com backoff exponencial, processamento assincrono, Dead Letter Queue (DLQ), auditoria de seguranca e metricas com Micrometer. Suporta multi-tenancy e arquitetura event-driven.

**Arquivos**: 48 classes Java
**Testes**: 3 classes de integracao (8 testes)
**Banco**: 5 tabelas com indices otimizados

---

## Arquitetura

### Estrutura de Pacotes

```
modules/webhooks/
├── domain/                    (5 entities)
│   ├── WebhookReceived.java
│   ├── WebhookSubscription.java
│   ├── WebhookDelivery.java
│   ├── WebhookDeliveryDlq.java
│   └── WebhookHmacFailureLog.java
├── repository/                (5 repositories)
├── internal/                  (19 services + handlers)
│   ├── WebhookSenderService.java
│   ├── WebhookReceiveService.java
│   ├── WebhookReceivedWorker.java
│   ├── WebhookRetryScheduler.java
│   ├── WebhookAdminService.java
│   ├── WebhookDlqService.java
│   ├── StripeWebhookHandler.java
│   ├── AsaasWebhookHandler.java
│   ├── IncomingWebhookHmacValidator.java
│   ├── WebhookSecurityProperties.java
│   ├── WebhookSecurityAuditService.java
│   ├── WebhookRateLimitInterceptor.java
│   ├── WebhookRateLimitConfig.java
│   ├── WebhookQueueHealthIndicator.java
│   ├── WebhookMetrics.java
│   ├── WebhookMetricsUpdater.java
│   └── ratelimit/
│       ├── RateLimitService.java (interface)
│       ├── InMemoryRateLimitService.java
│       └── RedisRateLimitService.java
├── web/                       (6 controllers)
│   ├── WebhookReceiveController.java
│   ├── WebhookAdminSubscriptionController.java
│   ├── WebhookAdminDeliveryController.java
│   ├── WebhookAdminReceivedController.java
│   ├── WebhookExceptionHandler.java
│   └── WebhookWebMvcConfig.java
├── api/dto/                   (1 DTO)
│   └── WebhookReceiveResponse.java
├── web/api/dto/               (3 DTOs)
│   ├── WebhookSubscriptionRequest.java
│   ├── WebhookSubscriptionResponse.java
│   └── WebhookDeliveryResponse.java
├── internal/events/           (2 event classes)
│   ├── IncomingAsaasWebhookEvent.java
│   └── IncomingStripeWebhookEvent.java
├── internal/listener/         (2 listeners)
│   ├── AsaasWebhookListener.java
│   └── StripeWebhookListener.java
├── internal/exception/        (2 exceptions)
│   ├── WebhookAuthException.java
│   └── WebhookPayloadTooLargeException.java
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
                                  |
                          [EXHAUSTED] ---> [DLQ (WebhookDeliveryDlq)]
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
| `errorMessage` | String(500) | Mensagem de erro |

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
- `EXHAUSTED` -> Esgotou tentativas (maximo atingido, movido para DLQ)

### 4. WebhookDeliveryDlq (Dead Letter Queue)

**Tabela**: `webhook_delivery_dlq`

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `id` | Long | PK auto-incremento |
| `deliveryId` | Long | ID original da delivery |
| `subscriptionId` | Long | ID da subscription |
| `empresaId` | Integer | Multi-tenancy |
| `eventType` | String(100) | Tipo do evento |
| `payload` | LONGTEXT | JSON do payload |
| `lastResponseStatus` | Integer | HTTP status da ultima tentativa |
| `lastResponseBody` | TEXT | Corpo da resposta (1000 chars) |
| `attemptCount` | Integer | Numero de tentativas realizadas |
| `reason` | String(255) | Razao do arquivamento |
| `createdAt` | LocalDateTime | Timestamp de arquivamento |

**Proposito**: Armazena deliveries EXHAUSTED para analise e possivel reprocessamento manual.

### 5. WebhookHmacFailureLog (Auditoria de Seguranca)

**Tabela**: `webhook_hmac_failure_log`

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `id` | Long | PK auto-incremento |
| `source` | String(50) | Source do webhook tentado |
| `ip` | String(100) | IP de origem |
| `userAgent` | String(255) | User-Agent do cliente |
| `message` | String(255) | Mensagem de erro |
| `createdAt` | LocalDateTime | Timestamp da falha |

**Proposito**: Registra todas as tentativas de webhook com HMAC invalido para analise de seguranca.

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
  "duplicate": false,
  "status": "PENDING"
}
```

**Codigos de Status**:
- `202 ACCEPTED` - Webhook recebido e validado
- `400 BAD REQUEST` - JSON invalido ou source invalido
- `401 UNAUTHORIZED` - HMAC invalido
- `413 PAYLOAD TOO LARGE` - Payload excede 1 MB
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

**Validacao de Payload Size**:
- Limite: 1 MB (1.048.576 bytes)
- Response: HTTP 413 (Payload Too Large)

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

### Rate Limiting (Strategy Pattern)

**Arquitetura**:
- **Interface**: `RateLimitService` (Strategy Pattern)
- **Implementacoes**:
  - `InMemoryRateLimitService` - Armazenamento local (default)
  - `RedisRateLimitService` - Armazenamento distribuido (Redis)
- **Config**: `WebhookRateLimitConfig` seleciona automaticamente via `@ConditionalOnBean`
- **Interceptor**: `WebhookRateLimitInterceptor` delega para o service

**Estrategias de Rate Limit**:

1. **InMemoryRateLimitService** (default):
   - Storage: `ConcurrentHashMap` em memoria
   - Chave: `{IP}|{source}`
   - Janela: 60 segundos (deslizante)
   - Thread-safe: `synchronized` per key
   - Limitacao: Reset ao reiniciar, sem compartilhamento entre instancias

2. **RedisRateLimitService** (producao distribuida):
   - Storage: Redis com INCR/EXPIRE
   - Chave: `webhook:ratelimit:{IP}|{source}`
   - TTL: 60 segundos
   - Compartilhado entre multiplas instancias
   - Resiste a restarts
   - Requer: `spring-boot-starter-data-redis` + `RedisTemplate<String, String>`

**Configuracao**:
```properties
# Limite de requisicoes por minuto
portalweb.webhooks.receiveRateLimitPerMinute=60

# Estrategia de storage (MEMORY ou REDIS)
portalweb.webhooks.rate-limit-store=MEMORY  # default

# Para usar Redis (alem de configurar o cliente):
portalweb.webhooks.rate-limit-store=REDIS
spring.data.redis.host=localhost
spring.data.redis.port=6379
```

**Fallback Automatico**:
- Se `rate-limit-store=REDIS` mas Redis nao disponivel: fallback para MEMORY
- Logs de advertencia para operador

### Input Validation

**Source**:
- Obrigatorio
- Max 50 caracteres
- Normalizado: lowercase, trim

**Payload**:
- Deve ser JSON valido
- Tamanho maximo: 1 MB (1.048.576 bytes)
- Armazenamento: LONGTEXT (suporta ate 4 GB)

**ExternalId**:
- Opcional
- Max 255 caracteres
- Usado para deteccao de duplicatas

### Auditoria de Seguranca

**WebhookSecurityAuditService**:
- Registra todas as falhas de HMAC
- Captura: source, IP, User-Agent, mensagem
- Armazena em `webhook_hmac_failure_log`
- Util para analise de tentativas de ataque

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
- DLQ integration (arquivamento de EXHAUSTED)
- Metricas de envio (sucesso/falha/duracao)

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
   - Se `attemptCount >= maxRetries`: Status `EXHAUSTED`, arquiva em DLQ
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
// Linha 116-119 de WebhookSenderService.java
double seconds = 30 * Math.pow(2, Math.max(0, currentAttempt - 1));
long delaySeconds = (long) Math.min(seconds, 43200); // cap 12h
delivery.setNextAttemptAt(LocalDateTime.now(UTC).plusSeconds(delaySeconds));
```

### DLQ Service (WebhookDlqService)

**Funcionalidade**:
- Arquiva deliveries EXHAUSTED
- Preserva informacoes de debugging
- Permite analise de falhas persistentes

**Metodo**:
```java
public void archive(WebhookDelivery delivery, String reason)
```

---

## Configuracao

### Application Properties

```properties
# Secrets para validacao HMAC (incoming)
portalweb.webhooks.incoming-secrets.stripe=whsec_stripe_secret_123
portalweb.webhooks.incoming-secrets.asaas=asaas_webhook_key_456

# Rate limiting
portalweb.webhooks.receiveRateLimitPerMinute=60
portalweb.webhooks.rate-limit-store=MEMORY       # MEMORY ou REDIS

# Redis (se rate-limit-store=REDIS)
spring.data.redis.host=localhost
spring.data.redis.port=6379

# Workers (intervalos em milissegundos)
portalweb.webhooks.worker-interval-ms=30000      # Worker processamento
portalweb.webhooks.retry-interval-ms=60000       # Scheduler retry
portalweb.webhooks.metrics-interval-ms=60000     # Atualizacao de metricas

# Health Check (opcional)
portalweb.webhooks.health.warning-threshold=500
portalweb.webhooks.health.critical-threshold=1000
```

### Valores Padrao (se nao configurados)

| Propriedade | Valor Padrao |
|------------|--------------|
| `receiveRateLimitPerMinute` | 60 |
| `rate-limit-store` | MEMORY |
| `worker-interval-ms` | 30000 (30s) |
| `retry-interval-ms` | 60000 (60s) |
| `metrics-interval-ms` | 60000 (60s) |
| `health.warning-threshold` | 500 |
| `health.critical-threshold` | 1000 |
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

**webhook_delivery_dlq**:
```sql
-- Sem indices adicionais (tabela de arquivo)
```

**webhook_hmac_failure_log**:
```sql
-- Sem indices adicionais (tabela de auditoria)
```

### Tamanhos de Colunas

| Coluna | Tipo | Limite |
|--------|------|--------|
| `payload` | LONGTEXT | 4 GB (MySQL), validado 1 MB |
| `headers` | LONGTEXT | 4 GB (MySQL) |
| `lastResponseBody` | TEXT | Truncado 1000 chars |
| `errorMessage` | String(500) | Truncado 500 chars |
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

### 9. Dead Letter Queue Pattern
Deliveries EXHAUSTED movidas para tabela DLQ.

### 10. Strategy Pattern (Rate Limiting)
Multiplas implementacoes (Memory/Redis) com selecao automatica.

### 11. Health Indicator Pattern
Custom health indicator para monitoramento de backlog.

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

**WebhookExceptionHandler**:
```java
log.warn("Webhook HMAC invalido. ip={} uri={} msg={}", ip, uri, ex.getMessage());
log.warn("Payload muito grande. ip={} uri={} msg={}", ip, uri, ex.getMessage());
```

### Metricas Implementadas (Micrometer)

**WebhookMetrics**:

**Counters**:
- `webhooks.incoming.received` - Total de webhooks recebidos
- `webhooks.incoming.received{source}` - Webhooks por source
- `webhooks.incoming.hmac_failures` - Falhas de HMAC
- `webhooks.incoming.payload_too_large` - Payloads rejeitados
- `webhooks.outgoing.send.success` - Envios bem-sucedidos
- `webhooks.outgoing.send.success{eventType}` - Sucesso por tipo
- `webhooks.outgoing.send.failure` - Envios falhados
- `webhooks.outgoing.send.failure{eventType}` - Falhas por tipo

**Gauges**:
- `webhooks.delivery.pending` - Deliveries pendentes
- `webhooks.received.pending` - Webhooks recebidos pendentes

**Timers**:
- `webhooks.outgoing.send.duration{eventType,status}` - Duracao dos envios

**WebhookMetricsUpdater**:
- Atualiza gauges a cada 60 segundos
- Evita consultas pesadas no banco a cada request

### Health Check (Actuator)

**WebhookQueueHealthIndicator**:

Expoe status de backlog em `/actuator/health` e `/actuator/health/webhooks`.

**Metricas Monitoradas**:
- `webhook_received.pending` - Webhooks recebidos aguardando processamento
- `webhook_delivery.pending` - Deliveries aguardando envio
- `webhook_delivery.failed` - Deliveries falhadas aguardando retry
- `webhook_delivery.exhausted` - Deliveries esgotadas (DLQ)

**Thresholds**:
```
Status UP:     backlog total < 500
Status WARNING: backlog total >= 500 e < 1000
Status DOWN:    backlog total >= 1000
```

**Response Example**:
```json
{
  "status": "UP",
  "components": {
    "webhooks": {
      "status": "UP",
      "details": {
        "receivedPending": 120,
        "deliveryPending": 45,
        "deliveryFailed": 10,
        "deliveryExhausted": 2,
        "totalBacklog": 175,
        "thresholds": {
          "warning": 500,
          "critical": 1000
        }
      }
    }
  }
}
```

**Configuracao**:
```properties
# Habilitar endpoint (ja vem habilitado em actuator)
management.endpoint.health.show-details=always
management.health.webhooks.enabled=true

# Thresholds customizados (opcional)
portalweb.webhooks.health.warning-threshold=500
portalweb.webhooks.health.critical-threshold=1000
```

**Uso Operacional**:
- Monitoramento: integrar com Prometheus/Grafana via `/actuator/prometheus`
- Alertas: configurar alertas se `status != UP`
- Load balancer: usar como readiness probe

### Recomendacoes

**Distributed Tracing** (TODO):
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

## Features Implementadas

✅ Recepcao de webhooks com HMAC-SHA256
✅ Rate limiting (60 req/min por IP+source)
✅ Validacao de payload size (1 MB)
✅ Deteccao de duplicatas (source + externalId)
✅ Processamento assincrono com worker
✅ Handlers para Stripe e Asaas
✅ Arquitetura event-driven
✅ Envio de webhooks (outgoing)
✅ Retry com backoff exponencial
✅ Dead Letter Queue (DLQ)
✅ Auditoria de falhas HMAC
✅ Admin APIs completas
✅ Metricas com Micrometer
✅ Multi-tenancy
✅ Rate limiting com Redis (Strategy Pattern)
✅ Health check de backlog (Actuator)

## Limitacoes Conhecidas

1. **Distributed Tracing**: Sem trace IDs. Considerar integracao com Sleuth/Zipkin para rastreamento distribuido.

---

## Referencias Tecnicas e Decisoes Arquiteturais

Abaixo, as referencias que fundamentaram as escolhas tecnicas deste modulo:

- **Java HttpClient** (Java 11+)
  - [Documentation](https://docs.oracle.com/en/java/javase/11/docs/api/java.net.http/java/net/http/HttpClient.html)
  - **Aplicacao:** Base do `WebhookSenderService`.
  - **Motivo:** Utilizacao de cliente nativo e assincrono para alta performance de I/O, eliminando a necessidade de dependencias externas pesadas (como Apache HttpClient ou OkHttp) para esta funcao.

- **HMAC-SHA256 for Webhook Security**
  - [RFC 2104 / Standard](https://datatracker.ietf.org/doc/html/rfc2104)
  - **Aplicacao:** Classes `IncomingWebhookHmacValidator` e `WebhookSenderService`.
  - **Motivo:** Garantia de integridade e autenticidade. Implementado com comparacao de tempo constante (*constant-time comparison*) para mitigar ataques de *timing*.

- **Webhook Best Practices**
  - [Webhooks.fyi](https://webhooks.fyi/best-practices/)
  - **Aplicacao:** Arquitetura geral do modulo.
  - **Motivo:** Adocao de padroes de mercado: resposta imediata (202 Accepted), processamento assincrono (Worker), idempotencia (`source` + `externalId`) e Dead Letter Queue (DLQ).

- **Exponential Backoff and Jitter**
  - [AWS Architecture Blog](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
  - **Aplicacao:** Logica de retentativa no `WebhookSenderService`.
  - **Motivo:** Evita o "efeito manada" e sobrecarga no servidor do cliente em caso de indisponibilidade, escalonando o tempo de espera progressivamente (30s, 1m, 2m... 12h).

- **Spring Events**
  - [Spring Documentation](https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html)
  - **Aplicacao:** `IncomingStripeWebhookEvent` e `IncomingAsaasWebhookEvent`.
  - **Motivo:** Desacoplamento total entre o recebimento do webhook e a regra de negocio. O modulo apenas publica o evento; outros modulos (Financeiro, Vendas) assinam e processam.

- **Spring Modulith**
  - [Reference Documentation](https://docs.spring.io/spring-modulith/reference/)
  - **Aplicacao:** Estrutura do pacote `com.auditoria.portalweb.modules.webhooks`.
  - **Motivo:** Garante fronteiras arquiteturais estritas, impedindo que classes internas do modulo sejam acessadas indevidamente por outras partes do sistema.

- **Micrometer Metrics**
  - [Micrometer Docs](https://micrometer.io/docs)
  - **Aplicacao:** `WebhookMetrics` e `WebhookMetricsUpdater`.
  - **Motivo:** Observabilidade em tempo real. Permite monitorar taxas de erro, falhas de HMAC e tamanho das filas de processamento sem impactar a performance.

---

**Autor**: Samuel Sereja Silva
**Versao**: 1.0.0 (Producao)
