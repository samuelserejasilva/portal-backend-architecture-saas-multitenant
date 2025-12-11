# 🧩 Guia de Desenvolvimento — Portal Auditoria 2.0 (Modulith)

**Última atualização:** 2025-10-29 08:30
**Contexto:** Spring Boot 3.5.7 • Java 21 • Spring Modulith 1.3.2 • MariaDB 12.0.2

> 📖 **Sobre este documento:**
> Este é o guia completo de desenvolvimento do projeto, consolidando boas práticas, padrões arquiteturais, convenções de código e histórico de alterações técnicas.
>  Documento para orientar o desenvolvimento do backend Java (Spring Boot / Spring Modulith / Multi-tenant).  
> Foco: **organização de módulos, segurança, multi-tenancy, banco de dados, compatibilidade e boas práticas de código.**

---

## 📚 **GUIA DE BOAS PRÁTICAS E PADRÕES DO PROJETO**

### 🚫 **O QUE NÃO FAZER**

#### **Dependências entre Módulos**

---

## 1. Visão geral da arquitetura

- **Stack principal**
  - Java 21+
  - Spring Boot 3.x (API REST, Security, Validation)
  - Spring Modulith (organização em módulos)
  - Hibernate / JPA
  - MariaDB (produção) / H2 (testes)
  - JWT (access/refresh tokens)
- **Modelo multi-tenant**
  - Shared Database / Shared Schema
  - Discriminador lógico `empresa_id` em todas as tabelas multi-tenant
  - Isolamento garantido na camada de aplicação:
    - JWT com `tenantId` + `role`
    - `AuthenticatedUser` no contexto de segurança
    - `TenantAccessFilter` validando vínculo `usuario_empresa`
    - Services/Repositories **sempre** filtrando por `empresa_id`

---

## 2. Padrões de módulos (Spring Modulith)

Este projeto é organizado em módulos sob `modules.*`, usando Spring Modulith.

### 2.1. Estrutura mínima de um módulo

Todo novo módulo deve seguir a convenção:

- Pacote raiz: `modules.<nome>`  
  Exemplos: `modules.corporate`, `modules.content`, `modules.webhooks`.

- Subpacotes recomendados:
  - `api` → controllers REST, DTOs expostos
  - `application` → services de orquestração / casos de uso
  - `domain` → entidades, agregados, regras de negócio
  - `spi` → interfaces/DTOs exportadas para outros módulos

### 2.2. O que TODO novo módulo deve ter

- [ ] Pacote próprio dentro de `modules` (`modules.<nome>`).
- [ ] `package-info.java` com as regras de dependência do Modulith.
- [ ] Pelo menos um `service` em `application` para encapsular regras de orquestração.
- [ ] Entidades e lógica de domínio em `domain` (não em controllers).
- [ ] Interfaces/DTOs para consumo externo em `spi` ou `api` (evitar expor entidades diretamente).
- [ ] Documento `docs/Modulos/IMPLEMENTACAO_MODULO_<NOME>.md` com:
  - Responsabilidade do módulo
  - Principais classes
  - Dependências com outros módulos
  - Decisões importantes / restrições

### 2.3. O que NÃO deve ser feito em um módulo

- ❌ Acessar diretamente:
  - Entidades JPA de outro módulo
  - Repositories de outro módulo
  - Services concretos de outro módulo  
  → Use interfaces em `spi` ou chamadas via `api` (DTOs) quando necessário.
- ❌ Criar dependência circular entre módulos:
  - Se `modules.content` depende de `modules.corporate`, então `modules.corporate` não pode depender de `modules.content`.
- ❌ Usar `@ComponentScan` ou configurações que “estouram” o boundary definido no Modulith.
- ❌ Ignorar falhas de testes de arquitetura (Modulith / ArchUnit):
  - Se quebrou, é porque alguma fronteira foi violada.

### 2.4. Fluxo recomendado para criar um novo módulo

1. Criar o pacote `modules.<nome>` com subpacotes `api/application/domain/spi`.
2. Definir a responsabilidade do módulo e rascunhar `IMPLEMENTACAO_MODULO_<NOME>.md`.
3. Expor apenas o estritamente necessário via `spi`/`api`.
4. Declarar dependências no `package-info.java`.
5. Rodar `./mvnw test` (incluindo testes Modulith/ArchUnit).
6. Só então criar controllers REST e endpoints públicos.

---

## 3. Multi-tenancy e segurança de dados

O backend é multi-tenant por aplicação. Alguns princípios são obrigatórios.

### 3.1. Modelo de dados

- Tabelas **multi-tenant** têm:
  - coluna `empresa_id` (NOT NULL)
  - FK → `empresas(id)`
  - índices por `empresa_id` (ex.: `idx_xxx_empresa`)
- `usuario`:
  - Tabela de identidade global (sem `empresa_id`).
  - Campos: `id`, `email`, `senha_hash`, `role` (global), `ativo`, timestamps.
- `usuario_empresa`:
  - Tabela de vínculo N:N entre `usuario` e `empresas`.
  - Campos principais:
    - `usuario_id`
    - `empresa_id`
    - `role` (contextual: `COMPANY_ADMIN`, `USER`, `AUDITOR`…)
    - `status` (`PENDING`, `ACTIVE`, `REVOKED`)
- `empresas`:
  - Informações da empresa (tenant)
  - Flag `publica_para_convite` para controles de busca pública.

### 3.2. Regras de OURO – o que SEMPRE fazer

- ✅ Em **TODO endpoint multi-tenant**, obter o tenant a partir do usuário autenticado:

  ```java
  @GetMapping
  public List<AlgoDTO> listar(
      @AuthenticationPrincipal(expression = "tenantId") Long tenantId,
      @AuthenticationPrincipal(expression = "role") String role
  ) { ... }
  ```

- ✅ Para **não-super_admin**, o `empresaId` efetivo deve SEMPRE vir do contexto (`tenantId`), nunca do cliente.
- ✅ Services/Repositories multi-tenant devem sempre usar métodos com `empresaId`, por exemplo:
  - `findByIdAndEmpresaId(id, empresaId)`
  - `deleteByIdAndEmpresaId(id, empresaId)`
  - `findAllByEmpresaId(empresaId)`
- ✅ `TenantAccessFilter` deve sempre:
  - Validar que o JWT tem `tenantId` (exceto super_admin em rotas globais)
  - Verificar se existe vínculo `usuario_empresa` com `status = ACTIVE` para `(usuario_id, empresa_id)`.
- ✅ Diferenciar papéis:
  - Global (`usuario.role`): ex. `super_admin`, `user`
  - Contextual (`usuario_empresa.role`): ex. `COMPANY_ADMIN`, `USER`, `AUDITOR`
  - `AuthenticatedUser.role`: papel efetivo no **tenant atual** (ou global, se SUPER_ADMIN).

### 3.3. O que NUNCA fazer (anti-padrões proibidos)

- ❌ Nunca confiar em `empresaId` enviado pelo cliente em:
  - `@RequestParam Integer empresaId`
  - `@PathVariable Integer empresaId`
  - `dto.getEmpresaId()`
- ❌ Nunca usar `findById(id)` / `deleteById(id)` em entidades multi-tenant sem incluir `empresaId`.
- ❌ Nunca expor `/api/v1/empresas/**` ou `/api/v1/pessoas/**` inteiras como públicas.
- ❌ Nunca gerar JWT ignorando `tenantId` ou `role` contextual.

### 3.4. Padrão recomendado para resolver empresaId no controller

Use sempre um padrão claro para resolver o `empresaId` efetivo:

```java
Integer resolveEmpresaId(String role, Long tenantId, Integer empresaIdFromRequest) {
    boolean superAdmin = "SUPER_ADMIN".equalsIgnoreCase(role) || "super_admin".equalsIgnoreCase(role);

    if (!superAdmin) {
        if (tenantId == null) {
            throw new AccessDeniedException("Tenant não definido para o usuário");
        }
        return tenantId.intValue(); // força tenant do contexto
    }

    // SUPER_ADMIN:
    // - se empresaIdFromRequest == null, pode listar todos (se fizer sentido)
    // - se != null, filtra pelo tenant informado
    return empresaIdFromRequest;
}
```

Controllers devem SEMPRE passar por essa regra (ou equivalente) antes de chamar services multi-tenant.

---

## 4. Padrões de segurança (auth, JWT, rotas públicas)

### 4.1. JWT e contexto de usuário

- Access Token:
  - Deve conter: `uid`, `email`, `tenantId`, `role` (contextual).
  - TTL curto (ex.: minutos).
- Refresh Token:
  - Contém pelo menos `uid`, `tenantId` (opcional dependendo da estratégia).
  - TTL mais longo.
- `JwtAuthenticationFilter` deve:
  - Validar assinatura e expiração.
  - Ler `uid`, `tenantId`, `role` do JWT.
  - Carregar o usuário do banco:
    - Confirmar se existe e está `ativo`.
  - Para SUPER_ADMIN:
    - reforçar o papel a partir do banco (não confiar só no token).

### 4.2. Rotas públicas x protegidas

- Rotas públicas típicas:
  - `/v3/api-docs/**`, `/swagger-ui/**`
  - `/actuator/health`, `/actuator/info`
  - `/api/v1/auth/**` (login, refresh, register, etc.)
  - Rotas específicas de conteúdo público (posts, serviços, layout/media) definidas no `TenantAccessFilter`.

- Tudo que envolver:
  - Dados de empresas
  - Dados de pessoas/usuários
  - Configurações (webhooks, integrações, etc.)  
  → deve exigir autenticação + validação de tenant.

### 4.3. Papéis e autorização

- Papéis globais (`usuario.role`):
  - `super_admin`, `user` (evitar misturar papel global com contextual).
- Papéis contextuais (`usuario_empresa.role`):
  - `COMPANY_ADMIN`, `USER`, `AUDITOR`, etc.
- `AuthenticatedUser.role` é o papel efetivo no contexto atual:
  - Usar esse valor em checks de autorização (`@PreAuthorize`, etc.).

---

## 5. Padrões de banco de dados e migrações

### 5.1. Convenções de nomenclatura

- Tabelas: `snake_case` (`usuario`, `usuario_empresa`, `webhook_delivery`).
- Colunas: `snake_case` (`empresa_id`, `created_at`, `senha_hash`).
- Constraints:
  - FKs: `fk_<tabela>_<referencia>` (ex.: `fk_usuario_empresa_usuario`)
  - UNIQUE: `uq_<tabela>_<campos>`
  - Índices: `idx_<tabela>_<coluna>`

### 5.2. Multi-tenancy

- Qualquer tabela que represente dado “por empresa” **deve ter**:
  - Coluna `empresa_id` NOT NULL
  - FK para `empresas(id)`
- Consultas SEMPRE devem incluir `empresa_id` (exceto em cenários globais bem controlados, ex.: super_admin).

### 5.3. Migrações

- Migrações são feitas via scripts SQL versionados (sem Flyway/Liquibase).
- Não criar/alterar tabelas diretamente no banco sem atualizar:
  - `schema.sql`
  - `data.sql`
  - `DB_SCHEMA_ONLY.sql` (para documentação)
- Toda mudança de schema deve ser registrada em:
  - `DB_CONTROL.md`
  - `DB_INDEX_MAP.md`
  - `db_change_checklist.md` (se aplicável).

---

## 6. Padrões de controllers, services, repositories e compatibilidade de código

### 6.1. Controllers

- Devem ser finos:
  - apenas validação básica, extração de contexto (`tenantId`, `role`), mapeamento de DTOs.
- Não colocar regras de negócio complexas em controllers.
- Em endpoints multi-tenant:
  - Sempre usar `tenantId` do usuário autenticado.
  - Ignorar/sobrescrever `empresaId` vindo do cliente (exceto super_admin).

### 6.2. Services

- Cada caso de uso principal deve ter um service correspondente.
- Services multi-tenant devem receber `empresaId` já resolvido pelo controller.
- Não acessar diretamente requisição HTTP ou SecurityContext dentro dos services (passar o que precisa, ex.: `empresaId`, `usuarioId`).

### 6.3. Repositories

- Para entidades multi-tenant:
  - Criar métodos específicos com `empresaId`:
    - `findByIdAndEmpresaId`
    - `deleteByIdAndEmpresaId`
    - `findAllByEmpresaId`
- Evitar uso de `findAll()` e `findById()` “nus” em entidades multi-tenant.

### 6.4. Convenções de código e compatibilidade (Beans, APIs modernas, etc.)

Esta seção resume **o que não usar** e **o que usar** na stack atual (Spring Boot 3.x, Java 21, JJWT novo, etc.), baseada nas confusões que já aconteceram.

#### 6.4.1. Beans de segurança (Spring Security 6+)

**Não usar mais:**

- `WebSecurityConfigurerAdapter` (deprecated/removido)
- Métodos antigos como:
  - `http.authorizeRequests()`
  - `http.csrf().disable()` encadeado no estilo antigo
  - `authenticationManagerBean()`

**Padrão correto:** usar Beans explícitos:

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}

@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf.disable())
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(PUBLIC_PATTERNS).permitAll()
            .anyRequest().authenticated()
        )
        .oauth2ResourceServer(oauth2 -> oauth2.jwt());
    return http.build();
}

@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.portalauditoria.com.br"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("*"));
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

#### 6.4.2. JWT – evitar API antiga do JJWT

**Não fazer (API antiga e confusa):**

- Usar apenas o jar `jjwt` “tudo em um” (0.9.x)
- Usar `SignatureAlgorithm.HS512` direto nos métodos estáticos antigos
- Misturar parsing antigo com criação nova

**Padrão correto:** (exemplo conceitual)

- Usar a nova forma (separando módulos `jjwt-api`, `jjwt-impl`, `jjwt-jackson`)
- Centralizar criação/parse de tokens em um `JwtTokenProvider` do projeto
- Nunca espalhar lógica de criação/parsing em controllers

#### 6.4.3. JPA Specifications

**Evitar:**
- Uso abusivo de `Specification.where(...)` encadeado com `and`/`or` de forma confusa
- Montar critérios “na mão” em cada repo

**Preferir:**
- Métodos helpers claros, por exemplo:

```java
public class EmpresaSpecs {
    public static Specification<Empresa> comNomeOuCnpj(String termo) {
        return (root, query, cb) -> {
            String like = "%" + termo.toLowerCase() + "%";
            return cb.or(
                cb.like(cb.lower(root.get("razaoSocial")), like),
                cb.like(cb.lower(root.get("cnpj")), like)
            );
        };
    }
}
```

- E usar em serviços:

```java
var spec = Specification.where(EmpresaSpecs.comNomeOuCnpj(filtro));
repo.findAll(spec);
```

#### 6.4.4. Streams e coleções (Java 21)

**Evitar (ainda funciona, mas é desnecessário):**

```java
list.stream().map(...).collect(Collectors.toList());
```

**Preferir:**

```java
list.stream().map(...).toList();
```

Menos ruído, mesma funcionalidade, API moderna.

---

## 7. Logs, auditoria e observabilidade

### 7.1. Logs

- Não logar:
  - senhas,
  - tokens JWT completos,
  - dados sensíveis em texto puro.
- Em operações críticas, logar:
  - `usuario_id`, `empresa_id`, ação, timestamp.
- Usar níveis adequados:
  - `INFO` para fluxo normal relevante,
  - `WARN`/`ERROR` para erros e situações anômalas,
  - `DEBUG` restrito para diagnóstico em desenvolvimento.

### 7.2. Auditoria

- Usar tabelas de auditoria (ex.: `audit_event` / `registro_auditoria`) para registrar ações importantes.
- Sempre incluir `empresa_id` na auditoria quando o evento for multi-tenant.

---

## 8. Testes

### 8.1. Tipos de testes

- **Unitários**:
  - Services, regras de negócio, helpers.
- **De integração**:
  - Controllers + services + repositories (com H2).
- **Modulith / ArchUnit**:
  - Garantir boundaries entre módulos.
- **Multi-tenancy** (obrigatório em endpoints sensíveis):
  - Usuário de empresa A não acessa dados de B.
  - `switch-tenant` altera realmente o contexto.

### 8.2. Checklist mínimo antes de commitar

- [ ] `./mvnw test` está passando.
- [ ] Não foi criado endpoint multi-tenant sem validação de tenant.
- [ ] Não foi adicionada rota pública sem ser intencional (ver `TenantAccessFilter`).
- [ ] Mudanças de banco foram refletidas nos scripts/documentação.

---

## 9. Checklist para criar um novo módulo

- [ ] Criou `modules.<nome>` com `api/application/domain/spi`?
- [ ] Definiu claramente a responsabilidade do módulo?
- [ ] Registrou `IMPLEMENTACAO_MODULO_<NOME>.md` em `docs/Modulos`?
- [ ] Configurou o `package-info.java` com as dependências permitidas?
- [ ] Evitou dependências diretas em entidades/repos/services de outros módulos?
- [ ] Rodou `./mvnw test` (incluindo testes Modulith/ArchUnit)?

---

## 10. Checklist para criar um novo endpoint multi-tenant

- [ ] Endpoint recebe `tenantId` via `@AuthenticationPrincipal(expression = "tenantId")`?
- [ ] Endpoint recebe `role` via `@AuthenticationPrincipal(expression = "role")` (se necessário)?
- [ ] NÃO confia em `empresaId` vindo do cliente (path, query, DTO)?
- [ ] Para não-super_admin, força `empresaId = tenantId`?
- [ ] Para SUPER_ADMIN, tratamento específico (pode ver todos / escolher empresa)?
- [ ] Service usa métodos de repository com `empresaId` (`findByIdAndEmpresaId`, etc.)?
- [ ] Não há `findAll()` sem filtro em entidade multi-tenant?
- [ ] Não expôs nada indevido em `PUBLIC_PATTERNS` (TenantAccessFilter)?
- [ ] Há testes cobrindo:
  - acesso permitido no tenant correto,
  - acesso negado ao tentar usar outro tenant?

---

## 11. Histórico de ajustes importantes

> Esta seção deve ser atualizada sempre que houver mudanças estruturais relevantes.

- **2025-12-11** – Endurecimento de multi-tenancy:
  - Correção do `JwtAuthenticationFilter` para usar `role` do JWT (contextual), com fallback seguro e reforço de `super_admin` pelo banco.
  - Revisão completa dos controllers multi-tenant para parar de confiar em `empresaId` vindo do cliente.
  - Ajustes em services/repositories (`findByIdAndEmpresaId`, `deleteByIdAndEmpresaId`, etc.) para reforçar o isolamento.
  - Redução de rotas públicas em `TenantAccessFilter` (remoção de wildcards amplos como `/api/v1/empresas/**` e `/api/v1/pessoas/**`).
- **2025-12-10** – Criação do `GUIA_DE_DESENVOLVIMENTO` unificado:
  - Consolidação das boas práticas de módulos, multi-tenant, segurança e compatibilidade de código.
  - Inclusão de seções “O que não fazer” (Beans antigos, JJWT legacy, Specifications confusas, Streams antigos).

---

> Este guia deve evoluir junto com o código.  
> Sempre que uma regra importante for descoberta (como as correções de multi-tenancy e segurança), atualize este documento para que novos desenvolvedores não repitam erros antigos.
