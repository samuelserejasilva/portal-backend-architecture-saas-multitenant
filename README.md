# PortalWeb - ERP & E-commerce Hub (SaaS Multi-tenant, API-first)

> Plataforma corporativa multi-tenant para gestão completa (ERP), automação financeira e hub de integração com grandes marketplaces (Amazon, B2W, Mercado Livre).

[![Java](https://img.shields.io/badge/Java-21-orange)](https://openjdk.org/projects/jdk/21/)
[![Spring Modulith](https://img.shields.io/badge/Architecture-Modular%20Monolith-blueviolet)](https://spring.io/projects/spring-modulith)
[![Architecture](https://github.com/samuelserejasilva/portal-backend-architecture-saas-multitenant/blob/main/ESTRUTURA-BAKEND.md)](#-visao-de-arquitetura)
[![Status](https://img.shields.io/badge/Status-Development-yellow)]()

---


O **PortalWeb** é um ecossistema de gestão empresarial (ERP) desenhado para alta performance. Ele unifica a gestão administrativa com a operação de vendas online, permitindo que empresas gerenciem desde o fluxo de caixa até a sincronização de estoques em múltiplos canais de venda.

A arquitetura segue o padrão **Modular Monolith**, garantindo que módulos complexos (como o Hub Integrador e Financeiro) operem de forma desacoplada e escalável.

---

## 📦 Módulos do Sistema

### 1. ⚡ [Módulo Webhooks (Core de Integração)](./bakend-modulo_webhooks.md)
O coração da comunicação com o mundo externo.
* **Função:** Receber notificações de vendas dos Marketplaces (Amazon, Americanas) em tempo real e disparar automações.
* **Tech:** Validação HMAC, Filas (Redis), Retry Inteligente e DLQ.
* **Status:** ✅ Enterprise Ready.

### 2. 🛒 Módulo Hub Integrador (Marketplaces)
Central de conexão com grandes players do e-commerce.
* **Funcionalidades:**
    * **Catalog Sync:** Envio de produtos para Amazon, Americanas (B2W), Magalu.
    * **Stock Sync:** Atualização de preço e estoque em tempo real para evitar "furo de estoque".
    * **Order Import:** Captura automática de pedidos via API.
 
    * ### 2.1 🛒 Módulo Hub Integrador (E-commerce & Dropshipping)
Central de inteligência para venda de produtos de terceiros (Marketplaces).
* **Catalog Import (Inbound):** Importação massiva de produtos da Amazon/B2W via API (Product Advertising API).
* **Price Intelligence:** Monitoramento em tempo real de preços e estoque na origem para precificação dinâmica no Portal.
* **Checkout Automation:**
    * *Modo Afiliado:* Geração de links traqueados (tag de parceiro).
    * *Modo Dropshipping:* Automação de pedidos na loja origem após confirmação de pagamento.

### 3. 💰 Módulo Financeiro (ERP)
Gestão completa do fluxo monetário da empresa.
* **Funcionalidades:**
    * **Contas a Pagar/Receber:** Controle de vencimentos e baixa automática.
    * **Conciliação:** Batimento automático de repasses dos marketplaces (taxas vs. líquido).
    * **DRE Gerencial:** Visão de lucro real por produto/canal.

### 4. 📦 Módulo de Vendas (OMS - Order Management)
Orquestração de pedidos centralizada.
* **Funcionalidades:** Fluxo de status do pedido (Aprovado -> Em Separação -> Faturado -> Enviado), emissão de NF-e e etiquetas de envio.

### 5. 📝 Módulo CMS & Institucional
Gestão de conteúdo para vitrines próprias ou portais corporativos.
* **Funcionalidades:** Gestão de Posts, Serviços e Páginas institucionais.

### 6. 🏢 Core Multi-tenant
Gestão de isolamento de dados e segurança.
* **Funcionalidades:** Gestão de Empresas (Tenants), Usuários, Permissões (RBAC) e Auditoria.

---

## 🚀 Tecnologias e Stack

* **Backend:** Java 21, Spring Boot 3 (Web, Security, Data JPA).
* **Arquitetura:** Spring Modulith (Event-Driven).
* **Banco de Dados:** MariaDB (Relacional) + Redis (Cache/Rate Limit).
* **Integrações:** Clientes HTTP robustos para Amazon SP-API e B2W API.
* **Observabilidade:** Micrometer + Spring Actuator.

---

## 🛠️ Como Executar

### Pré-requisitos
* JDK 21+
* Docker & Docker Compose (MariaDB + Redis)
* Maven

Autor: Samuel Sereja Silva
👤 Autor
Contador & Arquiteto de Software – Portal Auditoria 2.0
