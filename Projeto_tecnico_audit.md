# 📊 Projeto Técnico - Módulo Audit
## Plataforma SaaS Multi-tenant, API-first

**Versão:** 2.1.0
**Última Atualização:** 06/12/2025
**Status:** ✅ Backend 100%
**Arquitetura:** Plataforma SaaS multi-tenant, API-first

---

## 📋 Visão Geral

O módulo **Audit** é responsável por **auditoria de requisições HTTP e ações de domínio** no Portal Auditoria, implementando rastreabilidade completa através de interceptor HTTP e AOP para métodos de serviço.

### 🎯 Responsabilidades

- ✅ Auditoria automática de requisições HTTP (method, path, status, duration)
- ✅ Auditoria de ações de domínio via AOP (@Audited)
- ✅ Registro de actor (user_id, email), IP, user-agent, correlation-id
- ✅ Captura de payload antes/depois (arguments e result/error)
- ✅ Consulta administrativa paginada e filtrada
- ✅ Índices otimizados para queries por timestamp, actor e path

---

## 🏗️ Arquitetura do Módulo

### 📁 Estrutura de Diretórios

```txt
modules/audit/
├── package-info.java                    # @ApplicationModule (depend: users, auth, content, corporate, midia)
├── api/                                # Anotação AOP (1 arquivo)
│   └── Audited.java                    # ✅ @Audited para marcar métodos
├── domain/                             # Entidade JPA (1 arquivo)
│   └── AuditEvent.java                 # ✅ Entidade audit_event
├── repository/                         # Repositório JPA (1 arquivo)
│   └── AuditEventRepository.java       # ✅ JpaRepository + Specification
├── internal/                           # Serviços e Specs (4 arquivos)
│   ├── AuditService.java               # ✅ save() e search()
│   ├── aop/
│   │   └── AuditAspect.java            # ✅ @Aspect para @Audited
│   ├── spec/
│   │   └── AuditSpecs.java             # ✅ Filtros (path, actor, status, date)
│   └── web/
│       ├── HttpAuditInterceptor.java   # ✅ HandlerInterceptor
│       └── AuditWebMvcConfig.java      # ✅ Registra interceptor
└── web/                               # Controller Admin (1 arquivo)
    └── admin/
        └── AdminAuditController.java   # ✅ GET /api/admin/v1/audit

Total: 10 arquivos Java
```

### 🔗 Spring Modulith

```java
@ApplicationModule(
    allowedDependencies = {"users", "auth", "content", "corporate", "midia"}
)
```

---

## 🔧 Componentes Principais

### 1. @Audited (AOP Annotation)

Anotação para marcar métodos de serviço que devem ser auditados:

```java
@Audited(action="UPDATE", entity="Empresa", idParam="id")
public EmpresaDTO update(Long id, EmpresaUpdateDTO dto) { ... }
```

**Parâmetros:**
- `action`: Tipo de ação (CREATE, UPDATE, DELETE, etc.)
- `entity`: Nome da entidade
- `idParam`: Nome do parâmetro que contém o ID (opcional)
- `captureArgs`: Captura argumentos em payload_before (default: true)
- `captureResult`: Captura resultado em payload_after (default: true)

### 2. AuditAspect

Intercepta métodos anotados com `@Audited` e persiste eventos via AuditService:

- Captura actor do SecurityContext (AuthenticatedUser)
- Mede duração da execução
- Serializa args/result de forma segura (limite 5000 chars)
- Registra erros quando ocorrem exceções

### 3. HttpAuditInterceptor

Intercepta todas as requisições HTTP e registra:

- Method (GET, POST, etc.)
- Path (URI)
- Status (200, 404, 500, etc.)
- Duration (ms)
- Actor (email do SecurityContext)
- IP, User-Agent, Correlation-ID

**Exclusões:** `/actuator/**`, `/error`

### 4. AuditService

Serviço central para persistência e consulta:

```java
@Transactional
AuditEvent save(AuditEvent ev)

@Transactional(readOnly = true)
Page<AuditEvent> search(Specification<AuditEvent> spec, Pageable pageable)
```

### 5. AdminAuditController

API REST para consulta administrativa:

**Endpoint:** `GET /api/admin/v1/audit`

**Query Parameters:**
- `path` (String): Filtro por path (LIKE)
- `actorEmail` (String): Filtro por email (LIKE)
- `status` (Integer): Filtro por status HTTP
- `from` (LocalDateTime): Data início
- `to` (LocalDateTime): Data fim
- `page`, `size`, `sort`: Paginação

**Exemplo:**
```
GET /api/admin/v1/audit?actorEmail=admin@test.com&status=200&size=50&sort=createdAt,desc
```

---

## 🗄️ Banco de Dados

### Tabela `audit_event`

```sql
CREATE TABLE IF NOT EXISTS audit_event (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  actor_id BIGINT NULL,
  actor_email VARCHAR(160) NULL,
  ip VARCHAR(15) NULL,
  method VARCHAR(12) NULL,
  path VARCHAR(512) NULL,
  status INT NULL,
  duration_ms BIGINT NULL,
  action VARCHAR(80) NULL,
  entity_type VARCHAR(80) NULL,
  entity_id BIGINT NULL,
  user_agent VARCHAR(255) NULL,
  corr_id VARCHAR(64) NULL,
  payload_before LONGTEXT NULL,
  payload_after LONGTEXT NULL,
  empresa_id BIGINT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_uca1400_ai_ci;

CREATE INDEX idx_audit_ts ON audit_event(created_at);
CREATE INDEX idx_audit_actor ON audit_event(actor_id);
CREATE INDEX idx_audit_path ON audit_event(path);
```

**Campos principais:**

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `created_at` | DATETIME | Timestamp do evento |
| `actor_id` | BIGINT | ID do usuário responsável |
| `actor_email` | VARCHAR(160) | Email do usuário |
| `method` | VARCHAR(12) | HTTP method ou "AOP" |
| `path` | VARCHAR(512) | URI ou nome do método |
| `status` | INT | Status HTTP ou 200/500 |
| `duration_ms` | BIGINT | Tempo de execução |
| `action` | VARCHAR(80) | Tipo de ação (CREATE, UPDATE, etc.) |
| `entity_type` | VARCHAR(80) | Nome da entidade |
| `entity_id` | BIGINT | ID da entidade |
| `payload_before` | LONGTEXT | Args do método (JSON) |
| `payload_after` | LONGTEXT | Resultado ou erro (JSON) |
| `empresa_id` | BIGINT | Tenant ID (multi-tenant) |

---

## ⚙️ Configuração

### Propriedades

```properties
# Ativa/desativa o interceptor HTTP
app.audit.http.enabled=true
```

### Segurança

Proteja o endpoint admin no `SecurityFilterChain`:

```java
http.authorizeHttpRequests(a -> a
  .requestMatchers("/api/admin/v1/audit/**").hasRole("ADMIN")
);
```

### Dependência AOP

Adicione ao `pom.xml`:

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

---

## 📊 Uso Prático

### 1. Auditoria HTTP Automática

Todas as requisições HTTP são auditadas automaticamente (exceto `/actuator/**` e `/error`).

### 2. Auditoria de Métodos de Serviço

```java
@Service
public class EmpresaService {

  @Audited(action="CREATE", entity="Empresa")
  public EmpresaDTO create(EmpresaCreateDTO dto) {
    // Lógica de criação
  }

  @Audited(action="UPDATE", entity="Empresa", idParam="id")
  public EmpresaDTO update(Long id, EmpresaUpdateDTO dto) {
    // Lógica de atualização
  }

  @Audited(action="DELETE", entity="Empresa", idParam="id", captureResult=false)
  public void delete(Long id) {
    // Lógica de exclusão
  }
}
```

### 3. Consulta de Logs

```bash
# Listar logs do usuário admin nos últimos 7 dias
GET /api/admin/v1/audit?actorEmail=admin@test.com&from=2025-12-01T00:00:00

# Listar apenas erros (status 500)
GET /api/admin/v1/audit?status=500

# Buscar ações em path específico
GET /api/admin/v1/audit?path=/api/v1/empresas
```

---

## 🔒 Segurança e LGPD

### Boas Práticas

1. **Dados Sensíveis:** Evite anotar métodos que recebam senha, CPF, etc. Use `captureArgs=false` se necessário.

2. **Mascaramento:** Se precisar persistir payload em JSON, implemente Jackson Serializer customizado para mascarar campos sensíveis.

3. **Retenção:** Configure job para cleanup de logs antigos (ex: > 90 dias).

4. **Acesso Restrito:** Endpoint `/api/admin/v1/audit` deve ser acessível apenas por ADMIN.

---

## 📋 Integração com Outros Módulos

### Complementaridade

- **HttpAuditInterceptor:** Registra requisições HTTP (camada web)
- **AuditAspect:** Registra ações de domínio (camada service)
- **Rastreabilidade end-to-end:** Request → Controller → Service → Repository

### Exemplo de Fluxo Completo

```
1. HTTP Request: POST /api/v1/empresas
   → HttpAuditInterceptor registra: method=POST, path=/api/v1/empresas

2. Controller → Service: empresaService.create(dto)
   → AuditAspect registra: action=CREATE, entity=Empresa, payload_before=dto

3. Resultado:
   - 1 evento HTTP (request completa)
   - 1 evento AOP (ação de domínio)
   - Ambos com correlation-id para rastreabilidade
```

---

## 🎯 Resumo Final

**Status:** Módulo 100% funcional com auditoria HTTP e AOP

**Funcionalidades:**
- ✅ Interceptor HTTP automático
- ✅ Anotação @Audited para métodos
- ✅ AuditAspect com captura de args/result
- ✅ API admin com filtros e paginação
- ✅ Índices otimizados para consultas
- ✅ Multi-tenant ready (empresa_id)

**Integração:** users, auth, content, corporate, midia

---

**📅 Última Atualização:** 06/12/2025
**👥 Desenvolvido:** Equipe Portal Auditoria
**🏗️ Arquitetura:** Spring Modulith + AOP + HandlerInterceptor
**🌐 Versão:** 2.1.0
