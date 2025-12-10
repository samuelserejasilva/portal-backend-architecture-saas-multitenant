# Portal Backend – Architecture (SaaS Multi-tenant, API-first)

[![Java](https://img.shields.io/badge/Java-21-007396?logo=java&logoColor=white)](https://www.oracle.com/java/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.x-6DB33F?logo=springboot&logoColor=white)](https://spring.io/projects/spring-boot)
[![Spring Modulith](https://img.shields.io/badge/Spring%20Modulith-Modular%20Architecture-0F766E)](https://spring.io/projects/spring-modulith)
[![License: MIT](https://img.shields.io/badge/license-MIT-111827)](LICENSE)

**Plataforma SaaS multi-tenant, API-first**, construída com **Java 21 + Spring Boot 3 + Spring Modulith**, pensada para integrar com um frontend SPA de alta performance.

> ⚠️ **AVISO / DISCLAIMER**
>
> Este repositório é um **showcase de arquitetura backend**.
> O código de negócio real do Portal Auditoria é **proprietário / fechado**.
>
> Aqui você encontra:
>
> - Organização dos módulos backend (Spring Modulith)
> - Padrões de multi-tenant (empresa/tenant)
> - Implementação de Webhooks (entrada + saída, HMAC, fila, retry)
> - Governança de banco de dados e scripts de schema
> - Testes de integração com foco em arquitetura

---

## 🎯 Objetivo

Demonstrar como projetar um backend moderno para uma **Plataforma SaaS multi-tenant, API-first**, priorizando:

- 🧩 Arquitetura modular (bounded contexts bem definidos)
- 🔐 Segurança e isolamento por empresa (tenant)
- 🌐 APIs REST bem documentadas (OpenAPI)
- 📬 Webhooks robustos (incoming e outgoing)
- 🗄️ Governança de schema (nome de tabelas, FKs, JSON válido)
- ✅ Testes de integração garantindo o contrato dos módulos

Este repositório é o “par backend” do projeto:

- **Frontend Architecture:** `portal-frontend-architecture-vite-spa`  
  (SPA Vanilla TS + Vite, focada em performance)

---

## 🛠️ Stack

| Categoria     | Tecnologias                                             |
|--------------|---------------------------------------------------------|
| Linguagem    | Java 21                                                 |
| Framework    | Spring Boot 3.x                                         |
| Arquitetura  | Spring Modulith, camadas por módulo                     |
| Banco        | H2 (demo) / MariaDB (produção real)                     |
| Persistência | Spring Data JPA                                         |
| API          | Spring Web / Springdoc OpenAPI 3                        |
| Testes       | Spring Boot Test, Modulith Test, Testcontainers (opcional) |

---

## 🧩 Visão de módulos (Spring Modulith)

Estrutura proposta (pode variar um pouco do código final, mas esta é a ideia):

```text
backend/
├── src/main/java/com/example/portal/
│   ├── PortalBackendApplication.java
│   ├── config/
│   │   ├── SchedulingConfig.java      # @EnableScheduling, workers globais
│   │   ├── SecurityConfig.java        # Segurança simplificada (JWT/Basic)
│   │   └── OpenApiConfig.java         # Configuração do Springdoc
│   ├── modules/
│   │   ├── users/                     # Usuários e perfis (User/Role)
│   │   │   ├── domain/
│   │   │   ├── internal/
│   │   │   └── web/
│   │   ├── corporate/                 # Empresa (tenant)
│   │   │   ├── domain/
│   │   │   ├── internal/
│   │   │   └── web/
│   │   └── webhooks/                  # Módulo de Webhooks (incoming/outgoing)
│   │       ├── domain/
│   │       │   ├── WebhookSubscription.java
│   │       │   ├── WebhookDelivery.java
│   │       │   └── WebhookReceived.java
│   │       ├── internal/
│   │       │   ├── WebhookReceiveService.java
│   │       │   ├── WebhookSenderService.java
│   │       │   ├── WebhookReceivedWorker.java
│   │       │   └── handlers/          # handlers por source (Stripe, Asaas…)
│   │       └── web/
│   │           ├── WebhookReceiveController.java       # /api/v1/webhooks/receive/{source}
│   │           ├── WebhookAdminSubscriptionController.java
│   │           ├── WebhookAdminDeliveryController.java
│   │           └── WebhookAdminReceivedController.java
│   └── shared/                        # Exceções, DTOs comuns, utilitários
└── src/main/resources/
    ├── application.yml                # Configuração dev/demo
    ├── db/schema.sql                  # DDL simplificado
    └── db/data.sql                    # Dados de exemplo
O foco deste repositório é mostrar a arquitetura, não os detalhes de domínio real do Portal.

📬 Módulo Webhooks (SaaS-ready)
O módulo webhooks é o exemplo mais completo de uma funcionalidade SaaS multi-tenant, API-first.

Incoming Webhooks (recebimento)
Endpoint público:

POST /api/v1/webhooks/receive/{source}

Características:

Validação de HMAC via cabeçalho X-Webhook-Signature

Suporte a multi-tenant (empresaId via query/header)

Idempotência opcional via externalId

Persistência em webhook_received com status:

PENDING, PROCESSED, DISCARDED, FAILED

Worker (WebhookReceivedWorker) que:

lê PENDING,

roteia para handlers por source (Stripe, Asaas…),

marca como PROCESSED / FAILED / DISCARDED.

Outgoing Webhooks (envio)
Modelagem:

webhook_subscription: quem quer receber (URL, segredo, evento, retries, timeout)

webhook_delivery: fila + histórico de cada tentativa

Envio:

WebhookSenderService com java.net.http.HttpClient:

POST JSON

HMAC-SHA256 com secretKey da assinatura

Headers de auditoria (X-Webhook-Event-Type, X-Webhook-Delivery-Id)

Timeout configurável (timeoutSeconds)

Robustez:

status: PENDING, SENT, FAILED, EXHAUSTED

attemptCount, nextAttemptAt, lastResponseStatus, lastResponseBody

Backoff progressivo entre tentativas

Admin APIs
Endpoints administrativos (pensados para painel de controle ou SPA):

GET /api/v1/admin/webhooks/subscriptions

POST /api/v1/admin/webhooks/subscriptions

PUT /api/v1/admin/webhooks/subscriptions/{id}

GET /api/v1/admin/webhooks/deliveries

POST /api/v1/admin/webhooks/deliveries/{id}/retry

GET /api/v1/admin/webhooks/received

POST /api/v1/admin/webhooks/received/{id}/reprocess

Esses endpoints são consumíveis por qualquer frontend (SPA, painel interno, etc.), reforçando o conceito API-first.

🗄️ Banco de Dados & Governança
O repositório traz um exemplo simplificado de governança de schema:

Nomes em snake_case

Colunas de relacionamento padronizadas (*_id)

FKs nomeadas (fk_tabela_referencia)

Campos JSON (payload, headers) com CHECK (json_valid(...)) (MariaDB)

Campos de auditoria:

created_at, updated_at, processed_at, etc.

Scripts de exemplo:

src/main/resources/db/schema.sql

src/main/resources/db/data.sql

Em produção, o Portal real utiliza um banco MariaDB com collation utf8mb4_uca1400_ai_ci e controle de migração versionado; aqui mostramos uma versão reduzida e segura.

▶️ Como rodar (demo)
Requisitos mínimos:

Java 21+

Maven 3.9+

Banco H2 embutido (default) ou MariaDB local

bash
Copiar código
git clone https://github.com/samuelserejasilva/portal-backend-architecture-saas-multitenant.git
cd portal-backend-architecture-saas-multitenant/backend

# Rodar testes
./mvnw test

# Subir aplicação (perfil dev)
./mvnw spring-boot:run
Por padrão, a API sobe em:

http://localhost:8080

OpenAPI (se configurado):

http://localhost:8080/swagger-ui.html
ou

http://localhost:8080/swagger-ui/index.html

🌍 TL;DR (English)
This repository is a backend architecture showcase for a SaaS multi-tenant, API-first platform.

It demonstrates:

Java 21 + Spring Boot 3 + Spring Modulith modular design

Multi-tenant support via a corporate (company) module

A full Webhooks module:

incoming (HMAC, idempotency, workers),

outgoing (subscriptions, deliveries, retries)

Database governance and integration tests

Business logic is intentionally omitted – this is a technical portfolio.

👤 Autor
Samuel Sereja Silva
Contador & Arquiteto de Software – Portal Auditoria 2.0

GitHub: @samuelserejasilva

LinkedIn: https://www.linkedin.com/in/portalauditoria/

E-mail: samuel@portalauditoria.com.br

perl
Copiar código

