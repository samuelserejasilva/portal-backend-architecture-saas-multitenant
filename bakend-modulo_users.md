# 👤 Projeto Técnico - Módulo Users
## Plataforma SaaS Multi-tenant, API-first

**Versão:** 2.1.0
**Última Atualização:** 06/12/2025
**Status:** ✅ Backend 100%
**Arquitetura:** Plataforma SaaS multi-tenant, API-first

---

## 📋 Visão Geral

O módulo **Users** é responsável por **gestão de usuários do sistema** no Portal Auditoria, fornecendo cadastro, autenticação via BCrypt, controle de roles e API pública para registro.

### 🎯 Responsabilidades

- ✅ Cadastro de usuários (nome, email, senha hash)
- ✅ Validação de email único
- ✅ Hash de senha via BCrypt (PasswordEncoder)
- ✅ Controle de roles (ROLE_USER, ROLE_ADMIN)
- ✅ Gestão de status ativo/inativo
- ✅ API pública para registro
- ✅ API administrativa para CRUD completo
- ✅ Integração com módulo Auth (via repositório)

---

## 🏗️ Arquitetura do Módulo

### 📁 Estrutura de Diretórios

```txt
modules/users/
├── package-info.java                    # @ApplicationModule
├── api/                                # Interface pública (6 arquivos)
│   ├── package-info.java               # @NamedInterface("api")
│   ├── dto/
│   │   ├── UsuarioDTO.java             # ✅ record (público)
│   │   ├── UsuarioCreateDTO.java       # ✅ record (create)
│   │   └── UsuarioUpdateDTO.java       # ✅ record (update)
│   ├── mapper/
│   │   └── UsuarioMapper.java          # ✅ Entity → DTO
│   ├── UserPublicService.java          # ✅ Interface SPI
│   └── UserAuthDTO.java                # ✅ record (auth module)
├── domain/                             # Entidade JPA (1 arquivo)
│   └── Usuario.java                    # ✅ Entidade principal
├── repository/                         # Repositório JPA (1 arquivo)
│   └── UsuarioRepository.java          # ✅ JpaRepository
├── internal/                           # Implementações privadas (2 arquivos)
│   ├── UsuarioService.java             # ✅ CRUD + registro
│   ├── UserPublicServiceImpl.java      # ✅ Impl SPI
│   └── config/
│       └── UsersConfig.java            # ✅ PasswordEncoder bean
└── web/                               # Controllers REST (3 arquivos)
    ├── PublicUsuarioController.java    # ✅ POST /api/v1/users/register
    ├── UsersExceptionHandler.java      # ✅ Exception handling
    └── admin/
        └── AdminUsuarioController.java # ✅ POST/PUT/DELETE /api/v1/admin/users

Total: 16 arquivos Java
```

### 🔗 Spring Modulith

```java
@ApplicationModule(
    allowedDependencies = {"corporate::spi", "content::spi", "midia::api"}
)
```

**Named Interfaces:**
- `api` → Exporta: DTOs, Mappers, UserPublicService, UserAuthDTO

**Uso por outros módulos:**
```java
// Auth module
@ApplicationModule(allowedDependencies = {"modules.users :: api"})

// Acessa UsuarioRepository e PasswordEncoder
```

---

## 🗄️ Modelo de Dados

### Entidade `Usuario`

```java
@Entity
@Table(name = "usuario",
       uniqueConstraints = { @UniqueConstraint(name = "uk_usuario_email", columnNames = "email") }
)
public class Usuario {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  @Column(length=160, nullable=false)
  private String nome;

  @Column(length=160, nullable=false)
  private String email;                // Único, lowercase

  @Column(name="senha_hash", length=100, nullable=false)
  private String senhaHash;            // BCrypt hash

  @Column(length=40, nullable=false)
  private String role = "ROLE_USER";   // ROLE_USER | ROLE_ADMIN

  @Column(nullable=false)
  private Boolean ativo = true;

  @Column(name="empresa_id")
  private Long empresaId;              // Tenant ID (multi-tenant)

  @Column(name="created_at", nullable=false)
  private LocalDateTime createdAt = LocalDateTime.now();

  @Column(name="updated_at")
  private LocalDateTime updatedAt;
}
```

**Índices:**
- `uk_usuario_email` (unique): email
- `idx_usuario_email` (index): email (busca rápida)
- `idx_usuario_role` (index): role (filtros por role)

---

## 🔧 Componentes Principais

### 1. UsuarioService

Serviço central para todas as operações de usuários:

```java
@Service
public class UsuarioService {

  @Transactional
  public UsuarioDTO registrar(UsuarioCreateDTO dto) {
    // 1. Valida email único
    // 2. Converte email para lowercase
    // 3. Hash senha com BCrypt
    // 4. Define role (default: ROLE_USER)
    // 5. Persiste e retorna DTO
  }

  @Transactional
  public UsuarioDTO adminCreate(UsuarioCreateDTO dto) {
    // Mesmo que registrar (admin pode definir role)
  }

  @Transactional
  public UsuarioDTO adminUpdate(Long id, UsuarioUpdateDTO dto) {
    // 1. Busca usuário
    // 2. Atualiza campos não-nulos
    // 3. Hash nova senha se fornecida
    // 4. Atualiza updatedAt
  }

  @Transactional
  public void adminDelete(Long id) {
    // Remove usuário (soft delete futuro)
  }

  @Transactional(readOnly = true)
  public Page<UsuarioDTO> list(Pageable pageable) {
    // Lista paginada
  }

  @Transactional(readOnly = true)
  public UsuarioDTO get(Long id) {
    // Busca por ID
  }

  @Transactional(readOnly = true)
  public Usuario loadByEmail(String email) {
    // Para módulo Auth (credenciais)
  }
}
```

### 2. PasswordEncoder

Configurado em `UsersConfig`:

```java
@Configuration
public class UsersConfig {
  @Bean
  public PasswordEncoder passwordEncoder(){
    return new BCryptPasswordEncoder();
  }
}
```

**Uso:**
- `passwordEncoder.encode(senha)` → Hash para armazenar
- `passwordEncoder.matches(senha, hash)` → Valida no login

### 3. UsuarioMapper

Converte entidade para DTO público (não expõe senhaHash):

```java
@Component
public class UsuarioMapper {
  public UsuarioDTO toDTO(Usuario u) {
    // Converte todos campos exceto senhaHash
  }
}
```

---

## 🌐 API REST

### API Pública (Registro)

**Base Path:** `/api/v1/users`

| Método | Endpoint | Descrição | Request | Response |
|--------|----------|-----------|---------|----------|
| `POST` | `/register` | Registro público | UsuarioCreateDTO | UsuarioDTO (201) |

**Exemplo - Registro:**
```http
POST /api/v1/users/register
Content-Type: application/json

{
  "nome": "João Silva",
  "email": "joao@exemplo.com",
  "senha": "Senha123!",
  "role": "ROLE_USER"
}

---

HTTP/1.1 201 Created
{
  "id": 1,
  "nome": "João Silva",
  "email": "joao@exemplo.com",
  "empresaId": null,
  "role": "ROLE_USER",
  "ativo": true,
  "createdAt": "2025-12-06T10:30:00",
  "updatedAt": null
}
```

### API Administrativa (CRUD)

**Base Path:** `/api/v1/admin/users`

| Método | Endpoint | Descrição | Request | Response |
|--------|----------|-----------|---------|----------|
| `GET` | `/` | Lista paginada | Pageable params | Page<UsuarioDTO> |
| `GET` | `/{id}` | Busca por ID | - | UsuarioDTO |
| `POST` | `/` | Cria usuário | UsuarioCreateDTO | UsuarioDTO (201) |
| `PUT` | `/{id}` | Atualiza | UsuarioUpdateDTO | UsuarioDTO |
| `DELETE` | `/{id}` | Remove | - | 204 No Content |

**Exemplo - Listar:**
```http
GET /api/v1/admin/users?page=0&size=20&sort=createdAt,desc

{
  "content": [...],
  "pageable": {...},
  "totalElements": 100,
  "totalPages": 5
}
```

**Exemplo - Atualizar:**
```http
PUT /api/v1/admin/users/1
Content-Type: application/json

{
  "nome": "João Silva Jr",
  "novaSenha": "NovaSenha123!",
  "ativo": true
}

---

HTTP/1.1 200 OK
{
  "id": 1,
  "nome": "João Silva Jr",
  "email": "joao@exemplo.com",
  "role": "ROLE_USER",
  "ativo": true,
  "updatedAt": "2025-12-06T11:00:00"
}
```

---

## 📊 DTOs (Records)

### UsuarioDTO

```java
public record UsuarioDTO(
    Long id,
    String nome,
    String email,
    Long empresaId,
    String role,
    Boolean ativo,
    String createdAt,
    String updatedAt) {}
```

**Uso:** Todas as respostas (NÃO expõe senhaHash)

### UsuarioCreateDTO

```java
public record UsuarioCreateDTO(
    @NotBlank @Size(max=160) String nome,
    @NotBlank @Email @Size(max=160) String email,
    @NotBlank @Size(min=6, max=100) String senha,
    @Size(max=40) String role) {}
```

**Validações:**
- Nome: obrigatório, max 160 chars
- Email: obrigatório, formato válido, max 160 chars
- Senha: obrigatória, mínimo 6 chars, max 100 chars
- Role: opcional, max 40 chars (default: ROLE_USER)

### UsuarioUpdateDTO

```java
public record UsuarioUpdateDTO(
    @Size(max=160) String nome,
    @Email @Size(max=160) String email,
    @Size(min=6, max=100) String novaSenha,
    @Size(max=40) String role,
    Boolean ativo) {}
```

**Todos campos opcionais** (update parcial)

### UserAuthDTO

```java
public record UserAuthDTO(
    Long id,
    String email,
    String senhaHash,
    String role,
    Boolean ativo,
    Long empresaId) {}
```

**Uso:** Apenas para módulo Auth (contém senhaHash)

---

## 🔒 Segurança

### Validações

1. **Email único:** Constraint UK no banco + validação no service
2. **Email lowercase:** Sempre armazenado em lowercase
3. **Senha forte:** Mínimo 6 chars (recomendado: adicionar validação de complexidade)
4. **BCrypt hash:** Armazenado com custo 10 (padrão BCrypt)

### Controle de Acesso

**Configuração sugerida:**
```java
http.authorizeHttpRequests(auth -> auth
  .requestMatchers("/api/v1/users/register").permitAll()
  .requestMatchers("/api/v1/admin/users/**").hasRole("ADMIN")
  .anyRequest().authenticated()
);
```

### Boas Práticas

- **Nunca expõe senhaHash:** Apenas em UserAuthDTO (interno)
- **Update parcial:** Apenas campos informados são atualizados
- **Email imutável:** Recomendado (implementar validação)
- **Soft delete:** Futuro (manter ativo=false ao invés de DELETE)

---

## ⚙️ Configuração

### Propriedades

```properties
# BCrypt strength (opcional, padrão: 10)
# spring.security.bcrypt.strength=10
```

### DDL (MySQL/MariaDB)

```sql
CREATE TABLE IF NOT EXISTS usuario (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  nome VARCHAR(160) NOT NULL,
  email VARCHAR(160) NOT NULL,
  senha_hash VARCHAR(100) NOT NULL,
  role VARCHAR(40) NOT NULL DEFAULT 'ROLE_USER',
  ativo BOOLEAN NOT NULL DEFAULT TRUE,
  empresa_id BIGINT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NULL,
  CONSTRAINT uk_usuario_email UNIQUE (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_uca1400_ai_ci;

CREATE INDEX idx_usuario_email ON usuario(email);
CREATE INDEX idx_usuario_role ON usuario(role);
```

---

## 📋 Integração com Auth Module

O módulo Auth consome Users via:

```java
@Service
public class AuthService {
  private final UsuarioRepository usuarioRepo;  // ✅ Acesso direto
  private final PasswordEncoder passwordEncoder; // ✅ Bean compartilhado

  public TokenPair login(String email, String senha) {
    Usuario u = usuarioRepo.findByEmail(email.toLowerCase())
        .orElseThrow(() -> new IllegalArgumentException("credenciais inválidas"));

    if (!passwordEncoder.matches(senha, u.getSenhaHash())) {
      throw new IllegalArgumentException("credenciais inválidas");
    }

    // Gera JWT tokens...
  }
}
```

**Dependência Spring Modulith:**
```java
@ApplicationModule(allowedDependencies = {"modules.users :: api"})
package com.auditoria.portalweb.modules.auth;
```

---

## ⚠️ Melhorias Futuras

### 1. Validação de Senha Forte

Atualmente: apenas `@Size(min=6)`

Recomendado:
```java
@Pattern(
  regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]{6,}$",
  message = "Senha deve conter maiúscula, minúscula, número e caractere especial"
)
String senha;
```

---

## 🔮 Extensibilidades Futuras

### 1. Múltiplas Roles

Atualmente: `String role` (única)

Futuro:
```java
@ElementCollection
@CollectionTable(name = "usuario_roles")
private Set<String> roles;
```

### 2. Soft Delete

```java
@Column(name = "deleted_at")
private LocalDateTime deletedAt;

// Repository
List<Usuario> findByDeletedAtIsNull();
```

### 3. Perfil Completo

```java
@OneToOne(mappedBy = "usuario")
private UsuarioPerfil perfil;  // Foto, bio, preferências
```

### 4. Histórico de Senhas

```java
@OneToMany(mappedBy = "usuario")
private List<SenhaHistorico> senhasAnteriores;  // Evita reutilização
```

---

## 🎯 Resumo Final

**Status:** Módulo 100% funcional

**Funcionalidades:**
- ✅ Cadastro com validação de email único
- ✅ BCrypt hash de senha (PasswordEncoder)
- ✅ Controle de roles (ROLE_USER, ROLE_ADMIN)
- ✅ API pública (registro) - `/api/v1/users/register`
- ✅ API admin (CRUD completo) - `/api/v1/admin/users`
- ✅ DTOs usando records
- ✅ Integração com Auth module

**Integração:**
- Usado por Auth (UsuarioRepository, PasswordEncoder)
- Named Interface `api` expõe DTOs e UserPublicService
- PasswordEncoder compartilhado com Auth

**Qualidade:**
- ✅ DTOs são records (moderno)
- ✅ BCrypt hash (seguro)
- ✅ Email único (constraint UK)
- ✅ Validações Jakarta
- ✅ Paths seguem padrão do projeto (`/api/v1/admin/...`)
- ⚠️ Validação de senha básica (apenas tamanho mínimo)

---

**📅 Última Atualização:** 06/12/2025
**👥 Desenvolvido:** Equipe Portal Auditoria
**🏗️ Arquitetura:** Spring Modulith + BCrypt + Jakarta Validation
**🌐 Versão:** 2.1.0
