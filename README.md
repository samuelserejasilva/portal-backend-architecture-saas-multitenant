# Portal Auditoria 2.0 — Backend Engine (SaaS Multi-tenant)  API-first

> **Engine corporativa multi-tenant para gestão (ERP), automação financeira e hub de integração com marketplaces.**

[![Java](https://img.shields.io/badge/Java-21-orange?logo=openjdk)](https://openjdk.org/projects/jdk/21/)
[![Spring Modulith](https://img.shields.io/badge/Architecture-Modular%20Monolith-blueviolet?logo=spring)](https://spring.io/projects/spring-modulith)
[![Status](https://img.shields.io/badge/Status-Production%20Ready-brightgreen)](https://github.com/samuelserejasilva/portal-backend-architecture-saas-multitenant)
[![Security Score](https://img.shields.io/badge/Security%20Score-9.5%2F10-success?logo=owasp)](SECURITY.md)
[![OWASP Top 10](https://img.shields.io/badge/OWASP-Compliant-blue)](SECURITY.md)

---

## 🦅 Visão Geral

O **Portal Auditoria 2.0 Backend** é um ecossistema de alta performance desenhado para escalar. Ele unifica a gestão administrativa (ERP) com a operação de vendas online, suportando isolamento de dados rigoroso para múltiplos escritórios/empresas (Multi-tenancy).

A arquitetura segue o padrão **Modular Monolith** com **Spring Modulith**, garantindo que módulos complexos (como Webhooks e Integrações) operem de forma desacoplada, testável e segura, sem a complexidade prematura de microserviços.

---

## 🛡️ Segurança & Compliance (Score 9.5/10)

Este sistema foi auditado e classificado como **Top 5% do mercado** em segurança:

* ✅ **Autenticação:** JWT RS256 (Rotação automática) + MFA/2FA (TOTP, SMS, Email).
* ✅ **Isolamento Multi-tenant:** 4 camadas de proteção (Filter, Context, Worker, JPA).
* ✅ **Proteção:** Rate Limiting, Anti-Brute Force (4 layers) e Sanitização de Input.
* ✅ **Compliance:** OWASP Top 10 (10/10) e Zero Vulnerabilidades conhecidas.

---

## 📦 Módulos Principais

### 1. ⚡ [Core de Integração (Webhooks)](bakend-modulo_webhooks.md)
O coração da comunicação assíncrona.
* **Tech:** Validação HMAC SHA-256, Filas (Redis), Retry Inteligente (Exponential Backoff) e DLQ.
* **Capacidade:** Processamento resiliente de notificações de Marketplaces (Amazon, B2W).

### 2. 🛒 Hub Integrador
Central de conexão com grandes players do e-commerce.
* **Catalog Sync:** Sincronização de produtos e estoque em tempo real.
* **Price Intelligence:** Monitoramento de preços na origem para precificação dinâmica.

### 3. 🏢 Core Multi-tenant & IAM
Gestão de identidade e isolamento.
* **Estrutura:** Shared Database / Shared Schema com discriminador `empresa_id`.
* **Segurança:** Controle de acesso baseado em papel (RBAC) granular por Tenant.

---

## 🌐 Ecossistema do Projeto

Este backend é parte de uma solução completa:

| Componente | Repositório | Descrição |
| :--- | :--- | :--- |
| **Frontend** | [portal-frontend-architecture-vite-spa](https://github.com/samuelserejasilva/portal-frontend-architecture-vite-spa) | SPA de Alta Performance (Vanilla TS + Vite) |
| **Infraestrutura** | [Servidor-Windows-2022](https://github.com/samuelserejasilva/Servidor-Windows-2022) | Ambiente On-Premise Cloud-Native (IIS, SSL, Mikrotik) |

---

## 🚀 Tecnologias e Stack

* **Linguagem:** Java 21 LTS
* **Framework:** Spring Boot 3.x (Web, Security, Data JPA)
* **Arquitetura:** Spring Modulith (Event-Driven w/ Transactional Outbox)
* **Banco de Dados:** MariaDB (Persistência) + Redis (Cache/Rate Limit)
* **Observabilidade:** Micrometer + Spring Actuator

---

## 🛠️ Como Executar

### Pré-requisitos
* JDK 21+
* Docker Compose (para subir MariaDB + Redis)
* Maven

```bash
# 1. Subir dependências
docker-compose up -d

# 2. Executar aplicação
./mvnw spring-boot:run
👤 Autor: Samuel Sereja Silva Contador & Arquiteto de Software

